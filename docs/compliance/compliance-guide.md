# IRL Engine — Compliance Guide

*v1.1 · March 2026*

---

## Contents

1. [Regulatory Mapping](#1-regulatory-mapping)
2. [Cost of Non-Compliance](#2-cost-of-non-compliance)
3. [Liability Framework](#3-liability-framework)
4. [What IRL Proves and What It Does Not](#4-what-irl-proves-and-what-it-does-not)
5. [Sample Contract Clause](#5-sample-contract-clause)
6. [Audit Procedures](#6-audit-procedures)

---

## 1. Regulatory Mapping

IRL is designed to satisfy the audit trail and algorithmic governance requirements
embedded in each of the following frameworks. The table maps specific obligations
to the IRL capability that addresses them.

### MiFID II / MiFIR (EU)

| Obligation | Article / RTS | IRL capability |
|------------|--------------|----------------|
| Algorithmic trading system controls | Art. 17 MiFID II | Policy engine enforces pre-trade limits from the MTA; every decision is sealed before execution |
| Annual self-assessment of algorithmic systems | Art. 17(2) | `GET /irl/trace/:id` provides full reproducible audit of every decision |
| Kill-switch requirement | RTS 6, Art. 12 | `PATCH /irl/agents/:id/status {"status": "Suspended"}` — immediate, logged, cryptographically sealed |
| Clock synchronisation | RTS 25 | `txn_time` sourced from configurable time source; NTP-synced mode in Phase 2 |
| Audit trail for orders | RTS 24, Art. 3 | `reasoning_hash` + `final_proof` create a tamper-evident chain from intent to execution |
| Record retention (5 years) | RTS 24, Art. 1 | PostgreSQL audit table; export to long-term storage via SIEM pipeline |

### SEC Market Access Rule / Algorithmic Trading Proposals (US)

| Obligation | Rule | IRL capability |
|------------|------|----------------|
| Pre-trade risk controls | Rule 15c3-5 | Notional cap + regime-based direction control; enforced before order placement |
| Erroneous order prevention | SEC Concept Release on Equity Market Structure | Bitemporal constraint prevents stale reasoning; policy blocks out-of-regime directions |
| Activity monitoring and logging | Proposed Reg AT | Full trace JSON stored per decision; includes model hash, regime state, policy result |
| Kill-switch | Proposed Reg AT, §242.576 | Agent suspension via MAR; all subsequent authorize calls return `AGENT_NOT_ACTIVE` |

### DORA (EU Digital Operational Resilience Act)

| Obligation | Article | IRL capability |
|------------|---------|----------------|
| ICT risk management for financial entities | Art. 6 | IRL is a resilience layer — agent cannot trade without a sealed, policy-validated intent |
| Incident detection and reporting | Art. 10 | SIEM export of all policy violations; Prometheus alerts on anomalous block rates |
| Digital operational resilience testing | Art. 26 | Shadow mode enables safe policy testing without disrupting live operations |

### FINRA Algorithmic Trading (US)

| Obligation | Rule | IRL capability |
|------------|------|----------------|
| Written supervisory procedures for ATS | FINRA Rule 3110 | IRL trace provides the written, cryptographic record required for WSP documentation |
| Review of algorithmic strategies | Regulatory Notice 15-09 | Model hash enforcement ensures only registered, reviewed models can generate sealed intents |

---

## 2. Cost of Non-Compliance

These figures are illustrative reference points from public enforcement actions.
Your firm's exposure depends on jurisdiction, volume, and severity.

### Regulatory fines (historical benchmarks)

| Incident type | Jurisdiction | Fine range |
|--------------|-------------|------------|
| Market disruption by algorithmic system without adequate controls | US (SEC/CFTC) | $1M – $50M |
| Failure to maintain required audit trail for algorithmic orders | EU (NCA under MiFID II) | €500K – €5M |
| Inadequate pre-trade controls under market access rule | US (SEC Rule 15c3-5) | $1M – $15M |
| Flash crash contribution without kill-switch evidence | US/EU | $5M – $100M+ |

### Operational costs of an audit without IRL

When a regulator requests records for a specific algorithmic trading decision:

| Task | Without IRL | With IRL |
|------|------------|---------|
| Reconstruct agent state at decision time | Days to weeks (manual log archaeology) | Seconds (`GET /irl/trace/:id`) |
| Prove policy compliance at time of trade | Difficult — requires re-running simulation | Instant — `policy_result` and `policy_hash` are in the trace |
| Demonstrate kill-switch capability | Procedural; no cryptographic proof | Cryptographic — suspension event is in the audit log |
| Produce exchange correlation | Manual reconciliation across systems | `final_proof = SHA-256(reasoning_hash || exchange_tx_id)` |

### The ownership gap liability

Without IRL, when an autonomous agent causes a market incident:

> "We cannot determine whether the agent was authorized to make that trade
> under the regime conditions at that time."

This creates liability exposure for:
- The agent operator (fund)
- The model vendor (if inference was provided as a service)
- The exchange (if the order should have been rejected under their rules)

IRL closes this gap with a cryptographic chain: the `reasoning_hash` proves what
the agent knew and was permitted to do; the `final_proof` proves what the exchange
actually executed; and the bitemporal block proves when.

---

## 3. Liability Framework

### What the audit chain establishes

For each trade, the IRL audit chain establishes four facts:

1. **Identity:** The exact model (by hash) that produced the trade decision.
2. **Authorization:** The policy constraints that were in effect (by `policy_hash`),
   and whether the trade passed them (`policy_result = ALLOWED`).
3. **Epistemic state:** The agent's view of the market regime at decision time
   (`mta_regime_id`, `mta_version`, `mta_hash` — all signed by the MTA operator).
4. **Execution match:** Whether the exchange executed exactly what was authorized
   (`verification_status = MATCHED`) or something different (`DIVERGENT`).

### Liability allocation by scenario

| Scenario | Evidence | Liability allocation |
|----------|----------|---------------------|
| Trade was policy-compliant at decision time; exchange executed correctly | `policy_result=ALLOWED`, `verification_status=MATCHED` | Agent operator acted within authorized parameters |
| Trade was policy-blocked but executed anyway | `policy_result=HALTED`, exchange receipt present | No IRL-sealed trace for that trade — unauthorized execution |
| Model was not registered in MAR | `MODEL_HASH_MISMATCH` | Unauthorized model — operator failed to register; operator liability |
| Agent reasoned on stale regime data | `BiTemporalViolation` | IRL blocked the trade — clean evidence of control operating correctly |
| Exchange executed different quantity | `verification_status=DIVERGENT` | Exchange deviated from authorized intent; IRL proves the intent |

### What IRL does NOT prove

See §4 for a complete list. The critical limitation: IRL proves the agent's intent
and the policy check at the moment of sealing. It does not guarantee that the
agent's reasoning was *correct* — only that it was *authorized* under the policy
constraints in effect.

---

## 4. What IRL Proves and What It Does Not

### IRL proves

- The exact model (SHA-256 of config dict) that generated the trade decision.
- The regime state at decision time (signed by the MTA operator).
- The policy constraints that applied (regime-derived, not hardcoded).
- Whether the trade passed or failed policy at the time of sealing.
- The temporal relationship between agent reasoning and the system's receipt of
  the intent (`valid_time < txn_time`).
- Anti-replay: each heartbeat sequence is monotone; old heartbeats are rejected.
- Execution correlation: the `final_proof` cryptographically binds the reasoning
  hash to the exchange order ID.

### IRL does not prove

- That the agent's reasoning was correct or profitable.
- That the model's outputs were free of bias, hallucination, or error.
- That the exchange will execute what was authorized (exchange risk is external).
- That the `valid_time` accurately reflects when the model inference occurred —
  the agent provides this field; IRL enforces only that it is not in the future.
- That the MTA operator's regime signal is accurate (this is the MTA operator's
  responsibility and is governed by their separate attestation).
- Compliance with any regulation not explicitly addressed in §1 — IRL is an
  audit tool, not a regulatory compliance certification.

---

## 5. Sample Contract Clause

For inclusion in service agreements between AI agent vendors and fund operators,
or between fund operators and their prime brokers.

---

**Algorithmic Trading Audit Integrity**

The Operator shall maintain an IRL-compliant audit trail for all algorithmic
trading activity. Specifically:

(a) Every trade intent generated by an autonomous trading agent ("Agent") shall
    be sealed by the IRL Engine prior to order placement. The resulting
    `trace_id` and `reasoning_hash` shall be included in the order metadata
    submitted to the exchange.

(b) Every exchange execution report received by the Operator shall be submitted
    to the IRL Engine via the bind-execution endpoint within [60 minutes] of
    receipt. The resulting `final_proof` shall be stored with the execution record.

(c) The Operator shall maintain a Multi-Agent Registry (MAR) containing a
    current, accurate record of all Agent model hashes. No Agent shall be
    permitted to generate sealed intents whose `model_hash_hex` is not registered
    in the MAR.

(d) The Operator shall make available to [Counterparty / Regulator / Auditor]
    within [5 business days] of written request the full Reasoning_Trace_v1
    JSON for any trade identified by `trace_id` or by exchange order reference.

(e) The Operator shall maintain the IRL Engine audit database for a minimum of
    [5 years] from the date of each transaction, consistent with applicable
    record retention requirements.

(f) In the event of an IRL Engine outage, the Operator shall not place
    algorithmic orders during the outage period, or shall document and report
    such orders as exceptions to [Counterparty / Compliance Officer] within
    [24 hours] of the outage resolution.

---

*This clause is a starting-point template. Review with legal counsel before use.*

---

## 6. Audit Procedures

### Responding to a regulatory request for a specific trade

1. Obtain the exchange order ID or approximate timestamp from the regulator.
2. Query IRL for matching traces:
   ```sql
   SELECT trace_id, txn_time, reasoning_hash, final_proof, policy_result
   FROM irl.reasoning_traces
   WHERE exchange_tx_id = '<exchange_order_id>'
      OR client_order_id = '<internal_order_id>';
   ```
3. Fetch the full trace: `GET /irl/trace/<trace_id>` (or query `trace_json` column).
4. The response contains the complete `Reasoning_Trace_v1` including:
   - The model hash that produced the decision
   - The regime state and policy constraints at the time
   - The bitemporal timestamps
   - The cryptographic chain from reasoning to execution

### Annual algorithmic system self-assessment (MiFID II Art. 17)

1. Export all traces for the assessment period:
   ```sql
   SELECT *
   FROM irl.reasoning_traces
   WHERE txn_time BETWEEN :period_start AND :period_end;
   ```
2. Compute summary statistics:
   - Total intents sealed
   - Policy block rate (HALTED / total)
   - Divergence rate (DIVERGENT / total binds)
   - Orphan rate (EXPIRED / total intents)
3. For each active agent: confirm `model_hash_hex` matches the current registered
   model. If the model was updated during the period, verify a new registration
   record exists with the updated hash and a timestamp within the expected update window.
4. Confirm kill-switch capability: verify that agent suspension and re-activation
   events appear in the audit log, or execute a test cycle in the sandbox environment.

### Model change audit

When an agent model is updated:

1. Compute the new model hash: `IRLClient.compute_model_hash(new_config_dict)`.
2. Register the new hash in the MAR via `POST /irl/agents` (or update if allowed
   by your MAR governance process).
3. Verify the old model's last trace timestamp matches the expected cutover time.
4. Document the change in your model governance log with the old hash, new hash,
   and the IRL agent registration timestamp as the cryptographic evidence of
   when the change took effect.
