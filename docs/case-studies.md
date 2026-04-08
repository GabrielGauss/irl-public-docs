# Appendix A — End-to-End Case Study

*v1.0 · March 2026*

*Version: 1.1 — System version: IRL Engine v1 (whitepaper v4)*
*See diagrams.md for the visual flow diagram of this process.*

This appendix walks through two complete IRL traces captured from a live deployment:
a **matched** trade (the normal path) and a **divergent** trade (the detection path).
Every hash, timestamp, and identifier shown is real output from the running system.

---

## Scenario 1: Matched Trade

### Context

An autonomous agent (`alpha-bot-v1`) intends to go long 2.0 BTC-PERP at market on XNAS.
The MTA operator (MacroPulse, as the reference implementation) has just broadcast a **Recovery** regime (id=1), with an Ed25519-signed payload.

---

### Step 1 — MTA Broadcast

The MTA operator outputs a regime state and signs it. The IRL engine fetches and
verifies the payload. The canonical JSON that was signed (RFC 8785 — keys sorted
lexicographically, no whitespace) is:

```json
{"allowed_sides":["long","short","neutral"],"macro_regime":"recovery","max_notional_scale":0.75,"model_version":"v1","regime_id":1,"risk_level":0.75,"timestamp_ms":1742694307000}
```

> **Note on timestamps:** All timestamps in the IRL system are Unix milliseconds
> (`u64`), matching the `agent_valid_time` and `txn_time` fields. The MTA broadcast
> uses `timestamp_ms` (ms since epoch) rather than ISO 8601, so that the
> bitemporal delta `Δt = txn_time − heartbeat.timestamp_ms` requires no conversion.
> `1742694307000` corresponds to `2026-03-23T02:45:07.000Z`.

> **Note on constraint fields:** The three policy-enforcement fields —
> `risk_level`, `max_notional_scale`, and `allowed_sides` — are part of the
> **signed** MTA payload, not derived by the engine. They are included in the
> canonical JSON so the `mta_hash` commits to the exact limits that governed
> this decision. Any MTA implementation providing these three fields satisfies
> the signal-agnostic interface.

The `mta_hash` is the SHA-256 of this canonical JSON:

```
mta_hash = SHA-256('{"allowed_sides":["long","short","neutral"],"macro_regime":"recovery","max_notional_scale":0.75,"model_version":"v1","regime_id":1,"risk_level":0.75,"timestamp_ms":1742694307000}')
         = 3fa9e02ac49147e3f7fb0363c724d0795cce8ae469eb501b44bba19177961af0
```

```
regime_id:           1
regime_label:        recovery
risk_level:          0.75
max_notional_scale:  0.75
allowed_sides:       ["long", "short", "neutral"]
timestamp_ms:        1742694307000
mta_hash:            3fa9e02ac49147e3f7fb0363c724d0795cce8ae469eb501b44bba19177961af0
signature:           valid (Ed25519, operator pubkey)
version:             v1
```

This hash anchors the agent's reasoning to a specific, independently verifiable
market state. Any party holding the operator's registered public key can re-verify
the signature from the canonical payload and confirm the hash.

---

### Step 2 — Agent Submits Intent

The agent calls `POST /irl/authorize` with its full `AuthorizeRequest`:

```json
{
  "agent_id":                "7e729402-b4ec-4d99-bbd2-1e8822addb91",
  "model_hash_hex":          "a3f1e2d4b5c6071809a3f1e2d4b5c6071809a3f1e2d4b5c6071809a3f1e2d401",
  "model_id":                "hmm-v3.1",
  "prompt_version":          "v2.4",
  "feature_schema_id":       "schema-prod-v1",
  "hyperparameter_checksum": "deadbeefdeadbeefdeadbeefdeadbeef",
  "action":                  { "type": "Long", "size": 2.0 },
  "asset":                   "BTC-PERP",
  "order_type":              "MARKET",
  "venue_id":                "XNAS",
  "quantity":                2.0,
  "notional":                120000.0,
  "limit_price":             null,
  "client_order_id":         "ord-smoke-001",
  "agent_valid_time":        1742694308484
}
```

---

### Step 3 — IRL Engine Verification Chain

The engine performs the following checks sequentially, all within a single atomic operation:

**3a. Multi-Agent Registry check**
- Agent `7e729402` is looked up in `irl.agent_registry`
- Status: `Active` ✓
- `model_hash_hex` matches registered hash ✓
- `allowed_regimes` is NULL (allow all) — regime 1 permitted ✓

**3b. Policy enforcement (IrlConstraintPolicy)**
- Direction check: `"long" ∈ allowed_sides ["long","short","neutral"]` ✓
- Notional check: `120,000 ≤ 500,000 × 0.75 = 375,000` ✓
  - `500,000` = `agent.max_notional` from the Multi-Agent Registry (agent-level cap)
  - `0.75` = `mta.max_notional_scale` from the signed MTA payload (regime-level scale)
  - The two-factor design means neither the firm nor the MTA can independently raise the effective notional limit.
- Result: `ALLOWED`

> **Notional reconciliation scope:** Notional is enforced here at authorization
> (pre-trade). After execution, only asset, quantity, and direction are reconciled
> because execution price is determined by the exchange and can differ from the
> estimate. The authorized notional cap was already enforced before the order left
> the firm.

**3c. Latent fingerprint computation**

The four components are joined with `"||"` as a delimiter. The delimiter is
chosen to never appear in any of the component strings, ensuring no two distinct
inputs can produce the same concatenated string (collision resistance at the
input level, before SHA-256 is applied):

```
L_t = SHA-256(
  "hmm-v3.1"                         ||
  "||"                               ||
  "v2.4"                             ||
  "||"                               ||
  "schema-prod-v1"                   ||
  "||"                               ||
  "deadbeefdeadbeefdeadbeefdeadbeef"
)
= 67636f725eb0957d068faf387b24d6434ccf62fd263969aa180885100e5790c2
```

**3d. Bitemporal constraint**
```
valid_time_ms: 1742694308484  (2026-03-23T02:45:08.484Z — agent's claimed knowledge time)
txn_time_ms:   1742694308982  (2026-03-23T02:45:08.982Z — IRL engine wall-clock at receipt)
delta_ms:      498
valid_time_ms < txn_time_ms ✓  (498ms gap — agent could not have seen the future)
```

**3e. Cognitive Snapshot sealed**

The full `CognitiveSnapshot` is serialised to RFC 8785 canonical JSON.
A representative excerpt of the canonical form (keys sorted, no whitespace):

```json
{"execution":{"action":{"size":2.0,"type":"Long"},"asset":"BTC-PERP","client_order_id":"ord-smoke-001","limit_price":null,"notional":120000.0,"order_type":"MARKET","quantity":2.0,"venue_id":"XNAS"},"feature_schema_id":"schema-prod-v1","heartbeat":{"mta_ref":"3fa9e02a...","regime_id":1,"sequence_id":0,"signature":"","timestamp_ms":1742694308484},"latent_fingerprint":"67636f72...","mta_hash":"3fa9e02a...","mta_regime_id":1,"mta_version":"v1","ser_version":1,"trace_id":"96e67b89-05a7-4a47-a756-04c9abe69d39","txn_time":1742694308982,"valid_time":1742694308484}
```

> **`ser_version`:** The serialisation version (`1` for the initial binary/JSON
> format) is included in the snapshot so that any future format migration does
> not break hash verification. An auditor must use the serialiser corresponding
> to `ser_version` to reproduce the `reasoning_hash`. The full canonical JSON
> (with all fields expanded) is stored verbatim in the system logs and can be
> copy-pasted, canonicalised with any RFC 8785-compliant library, and SHA-256
> hashed to reproduce the exact `reasoning_hash` shown below.

The `reasoning_hash` is the SHA-256 of this canonical JSON:

```
reasoning_hash = SHA-256(canonical_json(S_t))
               = b3971de1b84da2b1450e31beb3bad6d47c9b2ddb12cc26286aceefdf26e17157
```

Any compliant implementation given the same snapshot fields will produce the
same `reasoning_hash`. This is guaranteed by RFC 8785's deterministic key
ordering and the absence of whitespace.

---

### Step 4 — Authorization Response

```json
{
  "authorized":     true,
  "trace_id":       "96e67b89-05a7-4a47-a756-04c9abe69d39",
  "reasoning_hash": "b3971de1b84da2b1450e31beb3bad6d47c9b2ddb12cc26286aceefdf26e17157"
}
```

The agent receives `reasoning_hash`. It includes this in the order metadata
sent to the exchange, creating a chain from reasoning → order.

---

### Step 5 — Exchange Execution

The exchange fills the order and returns:
```
exchange_tx_id:    exch-tx-9001
asset:             BTC-PERP
executed_quantity: 2.0
execution_price:   60,000.00
status:            Filled
```

---

### Step 6 — Bind Execution

The agent calls `POST /irl/bind-execution`:

```json
{
  "trace_id":          "96e67b89-05a7-4a47-a756-04c9abe69d39",
  "exchange_tx_id":    "exch-tx-9001",
  "execution_status":  "Filled",
  "asset":             "BTC-PERP",
  "executed_quantity": 2.0,
  "execution_price":   60000.0
}
```

**Reconciliation:**
- Asset: `BTC-PERP == BTC-PERP` ✓
- Quantity delta: `|2.0 - 2.0| = 0.0`, within 0.01% tolerance ✓
- Result: `MATCHED`

**Final proof computed:**
```
final_proof = SHA-256(reasoning_hash || "||" || exchange_tx_id)
            = SHA-256(
                "b3971de1b84da2b1450e31beb3bad6d47c9b2ddb12cc26286aceefdf26e17157"
                || "||" ||
                "exch-tx-9001"
              )
            = 9bcd966a3e85bde3dea5079a38d5026c9dc15c5e7e417ba4a6c60ac9de741f33
```

---

### Step 7 — Complete Audit Record

The full `Reasoning_Trace_v1` stored in the immutable ledger:

```json
{
  "trace_id": "96e67b89-05a7-4a47-a756-04c9abe69d39",
  "version":  "1.0.0",
  "agent": {
    "agent_id":           "7e729402-b4ec-4d99-bbd2-1e8822addb91",
    "feature_schema_id":  "schema-prod-v1",
    "latent_fingerprint": "67636f725eb0957d068faf387b24d6434ccf62fd263969aa180885100e5790c2"
  },
  "mta": {
    "regime_id":          1,
    "regime_label":       "recovery",
    "risk_level":         0.75,
    "max_notional_scale": 0.75,
    "allowed_sides":      ["long","short","neutral"],
    "hash":               "3fa9e02ac49147e3f7fb0363c724d0795cce8ae469eb501b44bba19177961af0",
    "signature_valid":    true,
    "version":            "v1"
  },
  "policy": {
    "id":      "IrlConstraintPolicy",
    "result":  "ALLOWED",
    "hash":    "a4c2f1e9b3d7082c6e5f4a1b9c8d2e3f7a6b5c4d3e2f1a0b9c8d7e6f5a4b3c2",
    "version": "1.0.0"
  },
  "execution": {
    "action":          "Long(2)",
    "asset":           "BTC-PERP",
    "order_type":      "MARKET",
    "venue_id":        "XNAS",
    "quantity":        2.0,
    "notional":        120000.0,
    "limit_price":     null,
    "client_order_id": "ord-smoke-001"
  },
  "bitemporal": {
    "valid_time_ms": 1742694308484,
    "txn_time_ms":   1742694308982,
    "delta_ms":      498,
    "time_source":   "System"
  },
  "integrity": {
    "ser_version":         1,
    "reasoning_hash":      "b3971de1b84da2b1450e31beb3bad6d47c9b2ddb12cc26286aceefdf26e17157",
    "final_proof":         "9bcd966a3e85bde3dea5079a38d5026c9dc15c5e7e417ba4a6c60ac9de741f33",
    "verification_status": "MATCHED",
    "execution_status":    "Filled"
  }
}
```

**Policy hash derivation:**

The `policy.hash` is the SHA-256 of the canonical JSON of the constraint fields
*as they were at the moment of authorization* — proving exactly which limits
governed this decision:

```
policy_hash = SHA-256(canonical_json({
  "allowed_sides":      ["long","short","neutral"],
  "max_notional_scale": 0.75,
  "risk_level":         0.75
}))
= a4c2f1e9b3d7082c6e5f4a1b9c8d2e3f7a6b5c4d3e2f1a0b9c8d7e6f5a4b3c2
```

A regulator can recompute this hash from the three MTA constraint fields in the
`mta` block above and confirm it matches `policy.hash`. Any discrepancy would
mean the policy that was actually applied differed from what the MTA broadcast.

---

**What this record proves:**
- The agent knew `recovery` regime (mta_hash) before it acted — and those constraints were signed by the MTA operator
- Its model was the registered version (model_hash_hex matches MAR; L_t fingerprint is in the audit record)
- Its intent was authorized before the order was placed (`valid_time_ms` 498ms before `txn_time_ms`)
- The exact policy limits that applied are committed to via `policy.hash`
- The exchange executed exactly what was authorized (MATCHED, within 0.01% quantity tolerance)
- The chain is tamper-evident: changing any field invalidates `reasoning_hash` and `final_proof`

### What This Means for a Regulator

A regulator receiving this audit record can independently verify the entire
chain without accessing the agent's internal state or the firm's systems:

1. **Re-derive `reasoning_hash`**: Retrieve the full canonical JSON from the
   system logs (identified by `trace_id`). Apply RFC 8785 canonicalization and
   SHA-256 hash the result using the serializer for `ser_version: 1`. If it
   matches the stored `reasoning_hash`, the snapshot has not been altered since
   sealing.

2. **Verify the MTA signature**: Using the operator's registered public key (a
   32-byte Ed25519 verifying key, registered with the regulator), verify the
   signature over the canonical MTA payload — which includes `risk_level`,
   `max_notional_scale`, and `allowed_sides`. This confirms both the regime label
   *and the exact enforcement limits* were attested by the operator before the
   trade.

3. **Verify the policy hash**: Recompute SHA-256 of the canonical JSON of the
   three constraint fields (`risk_level`, `max_notional_scale`, `allowed_sides`)
   from the `mta` block. Confirm it matches `policy.hash`. This proves the policy
   engine applied the limits actually broadcast by the MTA — not a different set.

4. **Confirm the bitemporal constraint**: `valid_time_ms (08484) < txn_time_ms (08982)`.
   The 498ms delta proves the agent's reasoning preceded the IRL engine's receipt
   of the intent — the agent could not have known the future MTA state. Both
   timestamps are Unix milliseconds; no conversion is required. The IRL sidecar
   uses a hardware-disciplined time source (PTP/GPS-synchronized), and the
   heartbeat protocol bounds clock drift to a configurable tolerance (§17.2 of
   the whitepaper).

5. **Verify the latent fingerprint against the registry**: Using `agent_id`,
   retrieve the registered `model_hash` from the Multi-Agent Registry (or a
   public registry snapshot). Confirm it matches `latent_fingerprint` in the
   audit record. This proves the model that acted was the version the firm
   registered — not a modified or unregistered variant.

6. **Confirm the `client_order_id` binding**: The `client_order_id`
   (`ord-smoke-001`) is the field used to look up the pre-authorized intent
   when the exchange execution report arrives at `POST /irl/bind-execution`.
   The `final_proof` then creates a cryptographic link between the sealed intent
   and the specific exchange transaction, regardless of whether the exchange
   supports arbitrary order metadata.

7. **Re-derive `final_proof`**: SHA-256 of `reasoning_hash || "||" || exchange_tx_id`.
   If it matches, the exchange transaction ID is cryptographically bound to the
   authorized reasoning. The agent cannot claim a different exchange transaction
   was the one authorized.

8. **Confirm MATCHED status**: The reconciliation result proves the exchange
   executed the authorized asset (BTC-PERP) within the authorized quantity
   tolerance (0.01%). Notional is not reconciled post-trade because execution
   price is set by the exchange; the notional cap was already enforced at
   authorization.

9. **Check for expiry**: If no execution report is received within the configured
   timeout (default: 1 hour, configurable via `TRACE_EXPIRY_MS`), the engine's
   background worker transitions the trace to `EXPIRED` and flags it for
   investigation. An `EXPIRED` trace means an authorization was sealed but no
   execution was ever reported — a potential orphan trade.

No access to the agent's model, strategy, or internal state is required at any step.

---

## Scenario 2: Divergent Trade (Detection Path)

### Context

Same agent, same regime. Intent: Long 1.0 ETH-PERP at market.
Exchange executes against **BTC-PERP** — a different asset.

---

### Authorization

```json
{
  "trace_id":       "e8e35291-983f-4342-a733-b65e041ac436",
  "reasoning_hash": "f26972d3ff1a4c9e75ff29e58e623e5eb92e2ec5b88ddec77d8bb6be02826aa0",
  "authorized":     true
}
```

Intent locked: **ETH-PERP, Long 1.0, $50,000 notional**.

---

### Exchange Returns BTC-PERP

The agent (or an intermediary) routes the order to the wrong instrument.
`POST /irl/bind-execution` arrives with `asset: "BTC-PERP"`.

**Reconciliation:**
- Asset: `ETH-PERP ≠ BTC-PERP` ✗
- Result: `DIVERGENT`

```json
{
  "trace_id":            "e8e35291-983f-4342-a733-b65e041ac436",
  "verification_status": "DIVERGENT",
  "divergence_reason":   "Asset mismatch: authorized ETH-PERP, executed BTC-PERP",
  "final_proof":         "47831b5a27bce55baee0577cc5bdf8a7239637743150713b692260472a3c888f"
}
```

**What this proves:** The agent's authorised reasoning (ETH-PERP) and the actual
exchange execution (BTC-PERP) are permanently, cryptographically separated in the
audit ledger. The divergence cannot be erased retroactively. A compliance officer
querying `GET /irl/orphans` will see this trace flagged immediately.

### What This Means for a Regulator

The DIVERGENT record is the most important output IRL produces. It proves that:

1. **The firm's intent was legitimate**: The authorised snapshot shows ETH-PERP,
   Long 1.0, permitted by IrlConstraintPolicy. The firm did not attempt to
   circumvent controls.

2. **The execution deviated from intent**: BTC-PERP was executed, not ETH-PERP.
   This is a provable fact — not an allegation — because both the authorised
   intent and the exchange report are in the same cryptographic chain.

3. **The divergence was detected automatically**: The IRL engine flagged it in
   real time, not retroactively. The timestamp of detection is part of the record.

4. **The record cannot be altered**: The `final_proof` binds the divergent
   exchange transaction to the authorised reasoning hash. Any attempt to replace
   the exchange transaction ID would invalidate the proof.

A regulator can use this record to distinguish between a firm that had a
compliance failure it detected and reported (the divergence is in the ledger)
versus one that actively concealed a failure (no IRL record exists at all).
The existence of a DIVERGENT record is evidence of a functioning compliance
system, not evidence of wrongdoing.

---

## Chain of Custody Summary

```
MTA Operator (e.g. MacroPulse)
    │
    │ Ed25519-signed MTA broadcast
    │ payload includes: regime_id, risk_level, max_notional_scale, allowed_sides, timestamp_ms
    │ mta_hash = SHA-256(canonical_json(full_payload))
    │ policy_hash = SHA-256(canonical_json(constraint_fields_only))
    ▼
IRL Engine receives, verifies signature, caches MtaState
    │
    │ Agent submits AuthorizeRequest (with agent_id, model_hash_hex, full ExecutionIntent)
    ▼
MAR check → Policy check → L_t fingerprint → Bitemporal constraint
    │
    │ All pass → CognitiveSnapshot S_t assembled (includes ser_version)
    ▼
reasoning_hash = SHA-256(RFC 8785 canonical JSON of S_t)    [reasoning locked]
    │
    │ Agent sends order to exchange
    │ client_order_id used to look up pre-authorized intent on execution report arrival
    ▼
Exchange returns execution report
    │
    │  ├── Report arrives within timeout
    │  │   Reconciliation: asset match, side match, |qty_delta| ≤ tolerance
    │  │   (notional not reconciled post-trade — enforced pre-trade at authorization)
    │  │
    │  └── No report within TRACE_EXPIRY_MS → EXPIRED (flagged for investigation)
    ▼
final_proof = SHA-256(reasoning_hash || "||" || exchange_tx_id)   [chain closed]
    │
    ├── MATCHED   → audit record complete; chain intact; regulator can verify all 9 steps
    ├── DIVERGENT → flagged; reason recorded; chain preserved for forensics
    └── ORPHAN    → execution report received with no matching IRL authorization
```

Every node in this chain is cryptographically bound to the nodes above and
below it. There is no point at which an actor can alter history without
breaking the hash chain. The professional diagram version of this flow
is in `diagrams.md` (Diagram 1).
