# IRL Engine — Incident Response Runbook

**Version:** 1.1
**Last updated:** 2026-04-02
**Owner:** Infrastructure / Compliance

This runbook covers the primary incident categories for IRL Engine.
For each scenario: detect → contain → eradicate → recover → post-mortem.

---

## Roles & Contacts

Fill in before going live. Every on-call engineer must be able to reach each role
within 5 minutes.

| Role | Responsibility | Contact |
|------|---------------|---------|
| Incident Commander | Coordinates response, owns comms | ___________ |
| Security Lead | Data breach, KMS, forensics | ___________ |
| DB Admin | PostgreSQL failover, WAL, backups | ___________ |
| MTA Operator (MacroPulse) | Regime outage, pubkey rotation | ___________ |
| Legal / Compliance | GDPR notifications, SOC 2 evidence | ___________ |
| Exchange Operations | Trade unwind, position reconciliation | ___________ |

---

## 1. Data Breach (Unauthorized Access to reasoning_traces)

### Indicators
- `admin_audit_log` rows with unrecognized `operator_id` or `ip_address`
- DB queries against `irl.reasoning_traces` not originating from `irl-engine`
- Alerts from PostgreSQL `pg_audit` extension (if enabled)

### Contain
1. Immediately rotate all `IRL_API_TOKENS` values — restart service with new tokens.
2. Revoke DB credentials: `ALTER ROLE irl_app PASSWORD '...'` and `REVOKE CONNECT`.
3. If mTLS is enabled, revoke the compromised client cert CN via CA CRL.
4. Snapshot the DB for forensic preservation:
   ```bash
   pg_dump --no-acl $DATABASE_URL > breach_snapshot_$(date +%Y%m%dT%H%M%S).sql
   sha256sum breach_snapshot_*.sql > breach_snapshot.sha256
   # Store both files in a write-once location (e.g. S3 with Object Lock enabled)
   ```
5. Capture system and network evidence before restarting anything:
   ```bash
   journalctl -u irl-engine --since "2 hours ago" > irl_engine_$(date +%Y%m%dT%H%M%S).log
   # If breach is ongoing: sudo tcpdump -w breach_$(date +%Y%m%dT%H%M%S).pcap -i eth0 port 5432
   ```

### Eradicate
1. Identify the blast radius: query `admin_audit_log` for the window.
2. Determine if any `trace_json` was read — check DB access logs.
3. If KMS key may be compromised: initiate key rotation (see §3).
4. Patch the entry vector (misconfigured firewall, leaked token, etc.).

### Recover
1. Restore service with new credentials and confirm clean audit log.
2. Notify affected firms within 72 hours per GDPR Art. 33 (if EU data involved).
3. File an incident report per SOC 2 requirements; run evidence export covering the
   breach window: `irl-engine-evidence-export --from <start> --to <end>`.

### Post-Mortem
- Root cause analysis within 5 business days.
- Update `allowed_mta_pubkeys` and IP allowlists as needed.

---

## 2. KMS Key Compromise

### Indicators
- HSM / Vault audit log shows unexpected `decrypt` calls.
- `irl-engine` logs `AppError::Encryption` on writes (not reads) — possible key reuse.
- AWS CloudTrail / Vault audit shows key material accessed outside expected principal.

### Contain
1. Mark the compromised key version as `retired` in `irl.kms_key_metadata`.
2. Set `KMS_KEY_VERSION` to the new version — new DEKs use the new CMK.
3. Do NOT delete the old key yet — existing traces must remain decryptable.

### Eradicate
1. Generate a new CMK in AWS KMS / Vault Transit.
2. Run the backfill job to re-encrypt all traces under the new key version.
3. After backfill completes and is verified: disable (not delete) the old CMK.
4. **Offline backups:** Any backup taken while the old key was active is still
   encrypted with that key. Before the old CMK is eventually deleted, either:
   - Re-encrypt all offline backups under the new CMK, **or**
   - Store the old CMK in an air-gapped, access-logged vault for the full data
     retention period (up to 7 years for audit logs). Document this in the
     key lifecycle record.

### Recover
1. Confirm all `reasoning_traces` rows have `key_version = NEW_VERSION`.
2. Confirm `kms_key_metadata` shows old version as `retired`.
3. Rotate all access credentials that had access to the old key.

### Post-Mortem
- Document how key material was exposed (over-broad IAM, leaked service account).
- Add SCP / Vault policy to limit decrypt principals.
- Consider reducing key scope (per-tenant CMKs) to limit blast radius of a future compromise.

---

## 3. Database Unavailability

### Indicators
- `irl-engine` returns `503` on `/irl/authorize` and `/irl/health`.
- `AppError::Database` in logs with `Connection refused` or `pool timed out`.
- Prometheus alert: `irl_authorize_total{result="database_error"}` non-zero.

### Contain
1. Check if the primary is reachable: `psql $DATABASE_URL -c '\l'`.
2. If primary is down, promote replica: `pg_ctl promote -D /data/pgdata`.
3. **Split-brain prevention:** Before promoting the replica, confirm the old primary
   is unreachable or shut it down explicitly. If the old primary recovers while the
   replica is already primary, you will have two writable databases. After promotion:
   - Take the old primary offline or reconfigure it as a read-only standby.
   - Never allow two writable primaries simultaneously.
4. Update `DATABASE_URL` to point to promoted replica and restart `irl-engine`.

### Recover
1. Confirm `irl.reasoning_traces` and `irl.admin_audit_log` are intact.
2. Re-run any pending pg_partman maintenance: `SELECT partman.run_maintenance_proc()`.
3. Verify read replica `DB_READONLY_URL` is still pointing to correct host.
4. Confirm no `reasoning_traces` rows have `verification_status = NULL` (orphaned).

### Post-Mortem
- Review connection pool config (`DB_POOL_MAX_CONNECTIONS`).
- Ensure DB failover is automatic (RDS Multi-AZ or streaming replica with Patroni).

---

## 4. MTA (Market Truth Anchor) Unreachable

### Indicators
- `irl-engine` logs `AppError::MtaFetchFailed` or `MTA unreachable — using last known regime`.
- After `FALLBACK_TTL_SECS` (default 60s): `POST /irl/authorize` returns `502 MTA_FETCH_FAILED`.
- Trading is halted (fail-closed) for agents that have not received a fresh regime.

### Contain
1. Identify MTA endpoint: `curl $MTA_URL/v1/regime/current`.
2. If MacroPulse is the operator: check MacroPulse status page.
3. If using a custom MTA: check operator's health endpoint and alert their on-call.

### Mitigate (while MTA is down)
1. If the firm has a fallback MTA operator configured in `allowed_mta_pubkeys`:
   - Update `MTA_URL` + `MTA_PUBKEY_HEX` to the fallback operator.
   - Restart `irl-engine` — trading resumes under fallback regime.
2. If no fallback: trading remains halted. This is the safe default.
3. **Multi-instance deployments:** Each IRL instance has its own fallback timer.
   When the MTA returns, instances may switch back at slightly different times,
   briefly creating inconsistent regime states across the fleet. To avoid this:
   - Trigger a coordinated rolling restart (`kubectl rollout restart` or equivalent).
   - Alternatively, expose a `/irl/admin/mta-refresh` endpoint (future work) that
     flushes the cached regime state across all instances simultaneously.

### Recover
1. Once primary MTA is restored: revert `MTA_URL` to primary, restart.
2. Confirm `POST /irl/authorize` succeeds and `mta_pubkey_used` in new traces matches primary.

### Post-Mortem
- Evaluate adding a second MTA operator (multi-MTA trust via `allowed_mta_pubkeys`).
- Review `FALLBACK_TTL_SECS` — shorter window = less stale regime risk; longer = more uptime.
  For highest compliance, consider reducing to 10–15 seconds or offering a `fail-immediate`
  mode (0s fallback) as a per-client option.

---

## 5. Agent Compromise (Rogue Agent)

### Indicators
- Agent presenting a valid token + correct model hash but anomalous trade patterns.
- `admin_audit_log` shows unexpected `AGENT_STATUS_CHANGE` or `TOKEN_ISSUE`.
- mTLS: cert CN matches but the cert file may have been exfiltrated.
- **Token leak (pre-use detection):** Unusual source IP or geographic location for
  an otherwise valid token; authorization rate spike from a single token outside
  normal trading hours. Query:
  ```sql
  SELECT DATE_TRUNC('minute', txn_time) AS minute, COUNT(*) AS reqs
  FROM irl.reasoning_traces
  WHERE agent_id = $1
    AND txn_time > NOW() - INTERVAL '1 hour'
  GROUP BY 1 ORDER BY 1;
  ```
  A sudden rate increase or off-hours spike warrants investigation before the attacker
  escalates. Periodic token rotation (e.g., every 30 days) limits the window of
  exposure even if a token is leaked.

### Contain
1. Immediately suspend the agent:
   ```
   PATCH /irl/agents/:id/status
   Body: {"status": "Suspended"}
   ```
2. The `admin_audit_log` records this action with operator identity.
3. Revoke all API tokens for the agent one by one (no bulk endpoint exists — list
   active tokens for the agent first, then delete each):
   ```
   DELETE /irl/admin/tokens/:token_id
   ```
4. If mTLS: add the agent cert serial to the CA CRL.

### Eradicate
1. Identify all traces from this agent in the incident window:
   ```sql
   SELECT trace_id, txn_time, policy_result
   FROM irl.reasoning_traces
   WHERE agent_id = $1 AND txn_time > $2;
   ```
2. Determine if any `AUTHORIZED` traces resulted in orders that should be unwound.
3. Notify risk management and exchange operations.

### Recover
1. If the agent binary is not compromised: re-register with a new `model_hash_hex`.
2. Issue new API token and (if mTLS) new client cert.
3. Set agent status back to `Active` after verification.

### Post-Mortem
- Review how agent credentials were exfiltrated.
- Consider adding `AGENT-01` (agent request signing) from v2 requirements.
- Implement alerting on per-token request rate (see Alerting Rules).

---

## 6. Performance Degradation (Latency Spike)

### Indicators
- `irl_authorize_duration_ms` p99 exceeds 200ms for more than 5 consecutive minutes.
- `irl_authorize_total{result="database_error"}` rising (pool saturation).
- Agents reporting timeouts or missed heartbeat deadlines.
- DB: `SELECT * FROM pg_stat_activity WHERE wait_event_type = 'Lock'` shows lock contention.

### Contain
1. Check current DB pool utilisation:
   ```sql
   SELECT count(*) FROM pg_stat_activity WHERE datname = 'irl';
   ```
2. If pool exhausted: temporarily raise `DB_POOL_MAX_CONNECTIONS` and restart
   `irl-engine` (hot config, no schema change needed).
3. If DB CPU is saturated: redirect analytics queries to `DB_READONLY_URL` and
   confirm no long-running transactions are holding locks.
4. If the Rust process is CPU-bound (check `top`/`htop`): scale horizontally by
   adding a second `irl-engine` instance behind the load balancer.

### Eradicate
1. Identify the root cause: DB lock contention, missing index, connection leak, or
   upstream I/O (MTA fetch latency regression).
2. Run `EXPLAIN ANALYZE` on the slowest query from `pg_stat_statements`.
3. If a code regression: roll back to the previous release tag.

### Recover
1. Confirm p99 returns below the SLA threshold for the operating concurrency tier.
2. Clear any orphaned connections: `SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE state = 'idle' AND query_start < NOW() - INTERVAL '5 minutes'`.

### Post-Mortem
- Add or tighten Prometheus alert thresholds (see Alerting Rules).
- Consider adding a Redis cache for MTA state to eliminate per-request MTA fetch
  latency under high concurrency.

---

## 7. Suspected Audit Tampering (Ledger Integrity Violation)

The entire trust model relies on tamper-evidence. If an attacker gains write access
to the DB and modifies historical traces, the cryptographic chain is broken.

### Indicators
- `verification_status` values changed to `MATCHED` on traces that were never bound.
- `reasoning_hash` in the DB does not match a recomputed hash of the stored
  `trace_json` (spot-check: fetch a trace via API, recompute SHA-256 over canonical
  JSON, compare).
- `pg_audit` entries missing for a time window (log deletion).
- WAL sequence gaps (unexpected LSN jumps in `pg_current_wal_lsn()`).
- If L2 anchoring is active: Merkle root stored externally does not match DB state.

### Contain
1. **Freeze writes immediately:** Put `irl-engine` into read-only mode or stop it.
   Do not run any further DB migrations or maintenance jobs.
2. Preserve the current DB state before anything is overwritten:
   ```bash
   pg_dump --no-acl $DATABASE_URL > tamper_snapshot_$(date +%Y%m%dT%H%M%S).sql
   sha256sum tamper_snapshot_*.sql > tamper_snapshot.sha256
   # Store in write-once / immutable location (S3 Object Lock, WORM storage)
   ```
3. Capture WAL logs for the suspected window: `pg_basebackup` or archive WAL
   segments covering the incident period.
4. Revoke all DB write credentials immediately.

### Eradicate
1. Compare every `reasoning_hash` in `irl.reasoning_traces` against a recomputed
   hash of its `trace_json`:
   ```sql
   -- Rows where stored hash diverges from expected are suspect.
   -- Recomputation must be done in application code (RFC 8785 canonical JSON + SHA-256).
   ```
2. Identify the attack vector: direct DB access, compromised migration tooling,
   or compromised operator credentials.
3. Restore clean data from the last known-good WAL point or backup.
4. Rotate all DB credentials, operator tokens, and KMS keys.

### Recover
1. Verify restored data against any external anchors (L2 Merkle roots, off-chain
   hash logs) before resuming trading.
2. Notify affected clients and regulators as required.

### Post-Mortem
- Enforce `pg_audit` on all DML against `irl.reasoning_traces` and alert on
  unexpected `UPDATE` or `DELETE` (these tables should be append-only in normal ops).
- Evaluate L2 continuous anchoring (external hash chain) to make future tampering
  detectable without DB access.

---

## Alerting Rules (Reference)

Define these in your Prometheus / Alertmanager config. Runbook section is indicated
for each alert.

| Alert | Expression | Severity | Runbook |
|-------|-----------|---------|---------|
| `IrlAuthorizeErrors` | `rate(irl_authorize_total{result="database_error"}[5m]) > 0` | P1 | §3 |
| `IrlHighP99Latency` | `histogram_quantile(0.99, rate(irl_authorize_duration_ms_bucket[5m])) > 200` | P1 | §6 |
| `IrlMtaFetchFailed` | `rate(irl_mta_fetch_failed_total[1m]) > 0` | P0 | §4 |
| `IrlDbPoolSaturated` | `irl_db_pool_waiting > 10` | P1 | §6 |
| `IrlTokenRateSpike` | `rate(irl_authorize_total[1m]) > 150` (per token label) | P2 | §5 |
