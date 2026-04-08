# IRL Engine — Service Level Agreement

**Version:** 1.1
**Effective:** 2026-04-02

---

## 1. Uptime Commitment

IRL Engine commits to **99.9% monthly uptime** for the `/irl/authorize` and
`/irl/bind-execution` endpoints.

| Metric | Target |
|--------|--------|
| Monthly uptime | ≥ 99.9% |
| Authorize p99 latency — L1 (≤ 100 concurrent agents, recommended hardware) | ≤ 50 ms |
| Authorize p99 latency — L1/L2 (≤ 500 concurrent agents, recommended hardware) | ≤ 150 ms |
| Authorize p99 latency — L2 (≤ 1 000 concurrent agents, recommended hardware) | ≤ 300 ms |
| Bind-execution p99 latency (non-hot path) | ≤ 200 ms |
| Maximum planned maintenance window | 30 minutes/month |
| Unplanned outage notification | ≤ 15 minutes |

**Recommended hardware for SLA applicability:** ≥ 8 vCPU, NVMe storage, PostgreSQL 15
tuned for write throughput (`synchronous_commit = local`, `shared_buffers ≥ 4 GB`),
co-located IRL Engine and DB (loopback or low-RTT LAN).

**Uptime calculation:** `(total_minutes − downtime_minutes) / total_minutes × 100`

**Definition of downtime:** Any consecutive 60-second window during which the
`/irl/authorize` endpoint returns 5xx errors for > 50% of requests.

**Scheduled maintenance** is excluded from downtime when notified ≥ 24 hours in advance.
Emergency patches applied outside scheduled windows **are counted as downtime** if they
cause service interruption.

---

## 2. Steady-State Definition

The p99 latency targets in §1 apply under **steady-state** conditions, defined as:

- Load is within the concurrency tier stated in §1.
- Per-token request rate does not exceed `RATE_LIMIT_PER_SECOND` (default: 100 req/s).
- Total aggregate request rate does not exceed 4 000 req/s (sustained).
- The IRL Engine process has been running for ≥ 60 seconds (past cold-start / DB pool
  warm-up).
- No concurrent DB maintenance operations (VACUUM FULL, index rebuild) on the
  `reasoning_traces` table.
- MTA is reachable and returning fresh regime data within 500ms.

The SLA does not cover latency during: cold starts, DB failover recovery windows, or
transient connection pool exhaustion events resolved within 30 seconds.

---

## 3. Client Responsibilities

The SLA guarantees above are contingent on the client:

- Deploying on hardware that meets the recommended specification in §1.
- Keeping agent clock drift within ± 200 ms of UTC (required by heartbeat validation).
- Rotating API tokens at least every 90 days.
- Not exceeding the per-token rate limit configured for their account.
- Using TLS 1.2+ for all connections to the IRL Engine endpoint.

---

## 4. Maintenance Windows

Planned maintenance is scheduled during low-trading hours:

- **Window:** Saturday 02:00–02:30 UTC
- **Notice:** ≥ 24 hours via status page and API header `X-IRL-Maintenance-At`
- **Scope:** DB migrations, TLS cert rotation, config changes

Emergency patches may be applied outside the window without notice when required
to remediate active security incidents. Such windows count as downtime (see §1).

---

## 5. Degraded-Mode Behaviour

IRL Engine degrades gracefully rather than failing open:

### MTA Unreachable
When the Market Truth Anchor endpoint is unreachable:
1. **Fallback window (0–60s):** Last verified regime state is used. Trading continues.
   All authorize responses include `mta_stale: true` in the trace metadata.
2. **After 60s:** Engine fails **closed** — `/irl/authorize` returns `502 MTA_FETCH_FAILED`.
   No trades are authorized without a fresh, signed regime broadcast.

> **Note:** The 60-second fallback window is configurable via `FALLBACK_TTL_SECS`.
> Clients with strict compliance requirements may reduce this to 10–15 seconds or
> request a `fail-immediate` mode (0s fallback) via contract addendum.

### Database Slow / Degraded
When DB response time exceeds 200ms:
1. Connection pool queue fills — new authorize requests return `503 DATABASE_ERROR`.
2. Existing in-flight traces complete normally.
3. `DB_READONLY_URL` analytics queries are isolated from the write-path pool.

### mTLS Certificate Expiry Warning
When any active client cert is within 14 days of expiry:
- `GET /irl/health` returns `cert_expiry_status: "WARNING"` with expiry dates.
- Authorize requests continue to succeed; no hard block until cert expires.

---

## 6. Incident Escalation

| Severity | Definition | Response SLA |
|----------|-----------|--------------|
| P0 | `/irl/authorize` down, trading halted | 15 min acknowledgement, 1 hr resolution target |
| P1 | Degraded latency (p99 > 200ms sustained) or partial outage | 30 min acknowledgement, 4 hr resolution target |
| P2 | Non-critical functionality impaired | Next business day |
| P3 | Documentation / cosmetic issues | Next sprint |

Higher-tier SLA packages (99.99% uptime, P0 resolution target 15 min) are available
under an Enterprise contract addendum.

---

## 7. Service Credits

If IRL Engine fails to meet the monthly uptime commitment, the client is entitled to
service credits applied against the following month's invoice:

| Monthly uptime achieved | Credit |
|------------------------|--------|
| 99.0% – 99.9% (exclusive) | 5% of monthly fee |
| 95.0% – 99.0% (exclusive) | 15% of monthly fee |
| Below 95.0% | 30% of monthly fee |

Credits are the sole remedy for uptime SLA breaches. Credits do not apply to latency
SLA breaches, scheduled maintenance windows, or exclusions listed in §9.

To claim a credit: submit a written request within 30 days of the incident, including
the affected time window and supporting metrics. Credits are forfeited if not claimed
within this period.

---

## 8. Data Retention

| Data | Retention |
|------|-----------|
| `irl.reasoning_traces` | Configurable via `DB_RETENTION_MONTHS` (default 60 months / 5 years) |
| `irl.admin_audit_log` | 7 years (SOC 2 requirement, immutable) |
| `irl.kms_key_metadata` | Indefinite (key audit chain must be permanent) |
| Application logs | 90 days |

> **MiFID II note:** Algorithmic trading records must be retained for 5 years under
> MiFID II Article 25. The default of 60 months satisfies this requirement. Clients
> subject to other regimes (e.g., CFTC 17 CFR Part 1.35: 5 years) should confirm
> the default is sufficient or extend via `DB_RETENTION_MONTHS`.

---

## 9. Exclusions

The SLA does not apply to:
- Force majeure events (network outages outside IRL Engine's infrastructure)
- Client-side issues (agent cert misconfiguration, invalid API tokens, clock drift > 200ms)
- Scheduled maintenance windows (notified ≥ 24h in advance)
- MTA operator outages beyond IRL Engine's control (see §5 degraded mode)
- Load exceeding the concurrency or rate-limit thresholds defined in §2

---

## 10. Compliance & Audit

IRL Engine is designed and built to meet SOC 2 Type II controls; a formal audit is
in progress. Evidence packages covering the current control set are available on
request via the `irl-engine-evidence-export` CLI. See
`docs/operations/incident-response.md` for incident response procedures.

Clients requiring a completed SOC 2 Type II report should contact their account
manager for availability timeline.

---

## 11. Monitoring & Reporting

- **Status page:** Available at the URL provided during onboarding. Updated within
  15 minutes of any P0 or P1 incident.
- **Monthly uptime report:** Delivered by the 5th of the following month, covering
  total uptime, incident count, and credits due (if any).
- **Metrics endpoint:** `GET /irl/metrics` (Prometheus format) exposes real-time
  latency histograms and error rates for client-side monitoring.
