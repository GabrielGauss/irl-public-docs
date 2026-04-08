# IRL Regulatory Mapping

IRL produces evidence that maps directly to specific regulatory requirements — not general "auditability," but the specific proof each framework asks for.

---

## EU AI Act

| Article | Requirement | IRL Mechanism |
|---|---|---|
| Art. 9 | Risk management system for high-risk AI | Policy Engine enforces per-agent risk controls pre-execution |
| Art. 12 | Record-keeping for high-risk AI systems | `reasoning_hash` = SHA-256(canonical CognitiveSnapshot); bitemporal ledger |
| Art. 13 | Transparency obligation | Forensic replay: sealed snapshot is fully reconstructable from trace_json |
| Art. 14 | Human oversight | Shadow mode: compliance officer reviews flagged decisions before enforcement |
| Art. 17 | Quality management | Multi-Agent Registry: model hash verification per agent |

---

## MiFID II / MiFIR

| Requirement | IRL Mechanism |
|---|---|
| Contemporaneous recordkeeping (Art. 25) | `txn_time` = IRL seal timestamp, `valid_time` = regime validity time — both immutable |
| Algorithmic trading identification (Art. 17) | `latent_fingerprint` = SHA-256(model_id ‖ prompt_version ‖ feature_schema_id) |
| Order audit trail | `client_order_id` + `exchange_tx_id` + `final_proof` link agent intent to exchange execution |
| Decision-maker field | `RegulatoryBlock.mifid_decision_maker`, `mifid_algo_id` |

---

## SEC Rule 15c3-5 (Market Access Rule)

| Requirement | IRL Mechanism |
|---|---|
| Pre-trade risk controls | Policy Engine blocks non-compliant intents before execution — hard halt, not a log |
| Capital threshold checks | Per-agent `max_notional` cap enforced at authorize time |
| Erroneous order prevention | `execution_action` + `quantity` validated against agent's allowed profile |

---

## CFTC Regulations

| Requirement | IRL Mechanism |
|---|---|
| CTI code tracking | `RegulatoryBlock.cftc_cti_code` on each trace |
| Account type classification | `RegulatoryBlock.cftc_account_type` |

---

## SEC CAT (Consolidated Audit Trail)

| Requirement | IRL Mechanism |
|---|---|
| Order identifier linkage | `RegulatoryBlock.cat_order_id` links IRL trace to CAT order record |

---

## DORA (Digital Operational Resilience Act)

| Requirement | IRL Mechanism |
|---|---|
| ICT risk management | Engine availability metrics via Prometheus; incident runbook included |
| Audit trail of system activity | Admin audit log: all agent status changes, token issuance, shadow-mode toggles |
| Third-party risk | IRL is self-hosted; no third-party data processor for audit records |

---

## Evidence Package for Regulators

A regulator requesting audit evidence for a specific trade needs:

1. `trace_id` — retrieve the full sealed `CognitiveSnapshot` via `GET /irl/trace/{trace_id}`
2. `reasoning_hash` — verify independently: `SHA-256(canonical_json(trace_json))` must match
3. `final_proof` — verify: `SHA-256(reasoning_hash ‖ exchange_tx_id)` must match
4. `verification_status` — `MATCHED` confirms execution conformed to stated intent

These four values are sufficient to demonstrate:
- The agent made a specific decision at a specific time based on specific inputs
- The decision was checked against policy and permitted
- The execution matched the authorized intent (or divergence was flagged)
- No post-hoc alteration is possible without breaking the SHA-256 chain
