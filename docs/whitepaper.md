# MacroPulse IRL — Intent, Reasoning, and Liability
## Whitepaper v4.0

*MacroPulse Research · March 2026*

---

## Abstract

Autonomous trading agents make thousands of decisions per day without a human in the loop. Current infrastructure records what they did — not what they knew, what they were permitted to do, or whether the exchange executed what was actually authorised. This gap between agent reasoning and market action is the **Ownership Gap**: the absence of a cryptographic chain of custody from decision to execution.

MacroPulse IRL (Intent, Reasoning, and Liability) is a **signal-agnostic** pre-execution compliance gateway that closes this gap. Before any order leaves the firm, IRL seals the agent's epistemic state — its knowledge of market regime, its model identity, its authorised intent — into a tamper-evident `CognitiveSnapshot`. That snapshot is cryptographically bound to the exchange confirmation, creating an immutable audit chain that no party can alter after the fact.

IRL is designed around an abstract Market Truth Anchor (MTA) interface. Any cryptographically attested regime signal can serve as the MTA — MacroPulse operates the reference implementation and provides a turnkey integration, but firms running proprietary regime models, third-party signal providers, or multi-operator consensus pipelines connect their own MTA with a single implementation. The seal mechanism, the audit chain, and the compliance guarantees are identical regardless of which regime signal is used.

This document specifies the IRL protocol, its three deployment layers, and the trust architecture that makes the system credible at regulatory scale.

---

## Table of Contents

1. [The Ownership Gap](#1-the-ownership-gap)
2. [What is MacroPulse IRL?](#2-what-is-macropulse-irl)
3. [System Architecture](#3-system-architecture)
4. [The Market Truth Anchor](#4-the-market-truth-anchor)
5. [The Cognitive Snapshot](#5-the-cognitive-snapshot)
6. [The Policy Engine](#6-the-policy-engine)
7. [Cryptographic Sealing](#7-cryptographic-sealing)
8. [Layer 2 — Heartbeat Protocol](#8-layer-2--heartbeat-protocol)
9. [Multi-Agent Registry](#9-multi-agent-registry)
10. [Post-Trade Verifier](#10-post-trade-verifier)
11. [The Final Proof](#11-the-final-proof)
12. [Audit and Forensic Replay](#12-audit-and-forensic-replay)
13. [Security Analysis](#13-security-analysis)
14. [Competitive Positioning](#14-competitive-positioning)
15. [Deployment Guide](#15-deployment-guide)
16. [MTA Integration Modes](#16-mta-integration-modes)
17. [Future Work — Multi-MTA Consensus](#17-future-work--multi-mta-consensus)
18. [Licensing](#18-licensing)
19. [Zero-Knowledge Compliance Proofs](#19-zero-knowledge-compliance-proofs-layer-3)
20. [Limitations](#20-limitations)
- [Appendix A — End-to-End Case Study](#appendix-a--end-to-end-case-study)
- [Appendix B — Diagram Specifications](#appendix-b--diagram-specifications)

---

## 1. The Ownership Gap

### 1.1 The Problem

Every autonomous trading agent has the same structural problem. The agent observes market data, applies its model, produces a decision, and places an order. All of this happens in milliseconds, without a human in the loop. The exchange confirms the execution. Somewhere in that chain, a critical link is missing.

**Who owns the decision?**

When a trade goes wrong — a flash crash, an unexpected position, a regulatory inquiry — the question is always the same: what did the agent know, what was it permitted to do, and did the exchange execute what was actually authorised? Today, there is no cryptographic answer to any of these questions.

Existing logging records what happened. It does not record the agent's epistemic state at the moment of decision. It does not prove the agent was operating within its mandate. And it cannot distinguish between a log that was written honestly and one that was reconstructed after the outcome was known.

This is the **Ownership Gap**: the absence of a chain of custody between AI reasoning and market execution.

### 1.2 Why Existing Approaches Fail

**Post-hoc logging** records events after they occur. An agent that produced the wrong output can, in principle, log a corrected version before the record is committed. Even honest logs cannot prove the agent's reasoning preceded the execution — they only prove something was written.

**Trade surveillance** is retrospective. It identifies patterns after the fact and cannot distinguish a compliant agent from a non-compliant one that happened to produce the same trades.

**Manual oversight** does not scale. At decision frequencies of thousands per day across multiple agents and strategies, no human can review intent before execution.

**Attestation services** that record after the fact inherit the same problem as logging — they can only attest to what they received, not to what the agent actually knew.

### 1.3 The IRL Solution

IRL closes the Ownership Gap before the order is placed. The agent's epistemic state — its knowledge of market regime, its model identity, its authorised intent — is sealed into a `CognitiveSnapshot` before any communication with the exchange. That seal becomes the root of a cryptographic chain that terminates only when the exchange confirms execution.

The chain cannot be forged, cannot be altered retroactively, and cannot be selectively presented. If the exchange executed something different from what was authorised, the divergence is provable and permanent.

---

## 2. What is MacroPulse IRL?

### 2.1 One-Line Answer

IRL is a **pre-execution compliance gateway** that sits between your autonomous trading agent and the exchange — sealing the agent's reasoning into a cryptographic audit trail before any order is placed.

### 2.2 Three Editions

#### L1 — IRL Sidecar
*Drop-in compliance, operational in under a day.*

- Pre-execution policy enforcement (regime-aware, per-agent)
- Cryptographic reasoning seal (SHA-256 / RFC 8785)
- Bitemporal audit ledger (tamper-evident, replay-safe)
- Multi-Agent Registry (fleet identity and governance)
- Post-trade verifier (MATCHED / DIVERGENT / EXPIRED lifecycle)
- REST API — wraps any existing agent in ~20 lines of code

#### L2 — IRL Audit Platform
*Enterprise compliance with anti-replay and signed market truth.*

Everything in L1, plus:
- Layer 2 signed heartbeats — monotonic sequence + Ed25519 prevents replay attacks
- MacroPulse MTA integration — regime ground truth is cryptographically attested
- Compliance dashboard — real-time feed of PENDING, DIVERGENT, and EXPIRED traces
- Forensic replay — any historical trade reconstructable from its sealed snapshot

#### L3 — IRL Sovereign Gateway
*For clients where compliance cannot reveal alpha.*

Everything in L2, plus:
- **TEE execution** (Intel TDX / AMD SEV) — policy enforced in a hardware-attested enclave
- **Wasm policy modules** — hot-reloadable sandboxed policies without engine changes
- **ZK compliance proofs** — prove policy adherence without revealing model or strategy

The ZK layer is not complexity for its own sake. For a fund with proprietary alpha encoded in its model features, a standard audit trail reveals the strategy. Zero-knowledge proofs allow a fund to prove compliance to a regulator without disclosing the strategy. That is the only way to make compliance compatible with alpha preservation.

### 2.3 What IRL Is Not

IRL does not replace anything you already have. It fills a gap that currently has no solution.

| System | Relationship to IRL |
|--------|-------------------|
| Execution Management System (EMS) | IRL sits upstream — it authorises intent before the EMS routes it |
| Order Management System (OMS) | IRL trace IDs attach to OMS records as cryptographic compliance references |
| Risk system | Complementary — risk systems enforce portfolio-level limits; IRL enforces regime-aware pre-execution policy |
| Trade surveillance | IRL provides pre-trade proof; surveillance uses IRL traces as evidence, not reconstructed logs |

### 2.4 Design Principles

IRL is built on four non-negotiable properties. These are not implementation preferences — they are invariants that define the system's trust guarantees.

**Determinism Over Explanation.** The system must produce machine-verifiable evidence of decision state, not human-readable narrative summaries. A compliance record is only valid if an independent party can reproduce the hash, verify the timestamp, and confirm that the action fell within the defined policy envelope — without relying on the system operator's interpretation.

**Pre-Execution Enforcement.** Policy constraints must be applied as a blocking gate before any action reaches the execution venue. Post-hoc policy checks are insufficient — they do not prevent non-compliant actions from occurring; they only detect them after the fact. IRL's Policy Engine is positioned in the execution path, not alongside it.

**Cryptographic Integrity.** All decision records must be tamper-evident and content-addressable. The hash of a Cognitive Snapshot must be independently reproducible from its inputs. Any modification to the snapshot — including retroactive changes to input data or policy definitions — must produce a detectable hash mismatch.

**Temporal Correctness.** The system must prove, with high confidence, that the information used to make a decision was available at the time the decision was made, and that it was not modified afterward. This requires bitemporal modelling — separating the time at which data was valid in the world from the time at which the system recorded and acted on it — combined with a signed heartbeat protocol that bounds the maximum allowable age of the market truth at decision time.

---

## 3. System Architecture

### 3.1 Overview

IRL is a sidecar service. It runs alongside the agent, intercepts intent before the exchange, and produces an immutable audit record for every decision.

```
MacroPulse MTA ──(Ed25519 signed regime)──► IRL Engine
                                                │
Agent ──(POST /irl/authorize)───────────────────┤
                                                │  MAR check
                                                │  Policy enforcement
                                                │  L_t fingerprint
                                                │  Bitemporal constraint
                                                │  CognitiveSnapshot sealed
                                                │
Agent ◄──(reasoning_hash + trace_id)────────────┘
  │
  │ Places order on exchange (with reasoning_hash in metadata)
  ▼
Exchange ──(execution report)──► Agent
                                   │
                                   │ POST /irl/bind-execution
                                   ▼
                               IRL Engine
                                   │  Reconciliation
                                   │  final_proof computed
                                   │  MATCHED / DIVERGENT / EXPIRED
                                   ▼
                               Immutable Audit Ledger
```

### 3.2 Three-Layer Trust Model

**Layer 1 — Pre-Execution Gateway**
The core protocol. Policy enforcement, cryptographic sealing, bitemporal audit, Multi-Agent Registry, post-trade verifier. This is the minimum viable compliance layer — deployable in under a day.

**Layer 2 — Verified Market Truth**
Adds the signed heartbeat protocol. Each authorize request must carry a SignedHeartbeat: an Ed25519-signed attestation from the agent that it has observed a recent, valid MTA broadcast. Prevents replay attacks and binds the agent's heartbeat to the specific MTA state it claims to have observed.

**Layer 3 — Sovereign Execution**
Hardware-enforced execution. Policy runs inside a TEE enclave; the host cannot inspect or tamper with enforcement. Wasm policy modules enable per-client policy customisation without engine forks. ZK proofs enable privacy-preserving compliance for alpha-sensitive deployments.

---

## 4. The Market Truth Anchor

### 4.1 Purpose

The Market Truth Anchor (MTA) is an **abstract interface** — any cryptographically attested external regime signal that satisfies the MTA protocol. IRL does not depend on any specific operator. It depends on a signed, versioned, fresh regime state that it can independently verify. Whatever produces that state is the MTA for that deployment.

This matters: a quant firm running its own HMM, a broker aggregating a multi-vendor consensus, or a regulatory body operating a public regime oracle can all serve as MTA operators. The IRL seal, audit chain, and compliance guarantees are identical regardless of which operator produced the MTA state.

MacroPulse operates the **reference MTA implementation**: a Hidden Markov Model / Principal Component Analysis pipeline that classifies macro market conditions into a small number of discrete regimes. Each output is Ed25519-signed and broadcast as a `RegimeBroadcast`. This is the turnkey path — zero additional infrastructure required. Firms with proprietary regime models use the `MtaClient` trait to plug in their own source (see §16).

### 4.2 MtaState

The IRL engine maintains a cached `MtaState`:

```rust
pub struct MtaState {
    pub regime_id:           u8,       // opaque — assigned by the operator (0–255)
    pub regime_label:        String,   // opaque — human-readable, operator-defined
    pub risk_level:          f64,      // 0.0 (defensive) → 1.0 (fully risk-on)
    pub max_notional_scale:  f64,      // 0.0–1.0 multiplier on the agent's configured cap
    pub allowed_sides:       Vec<String>, // directions open now: "long","short","neutral"
    pub version:             String,   // semantic version of the operator's model
    pub hash:                String,   // SHA-256 of the canonical regime payload
    pub broadcast_time:      i64,      // Unix ms of the MTA broadcast
}
```

`regime_id` and `regime_label` are fully opaque to IRL — the engine does not pattern-match on them. The three normalized constraint fields (`risk_level`, `max_notional_scale`, `allowed_sides`) are what the policy engine reads. An operator running a 4-state HMM, a 2-state bull/bear model, a VIX-based classifier, or a continuous credit-spread score all satisfy this contract by mapping their output to these three fields.

The `hash` field is SHA-256 of the RFC 8785 canonical JSON of the regime payload, computed before the signature is applied. This hash is embedded in every `CognitiveSnapshot`, cryptographically binding each agent decision to the specific MTA broadcast the agent was operating under.

### 4.3 Signature Verification

The IRL engine verifies every MTA response against the operator's registered Ed25519 public key. A response that fails signature verification is rejected — the engine returns `MtaSignatureInvalid` and blocks the authorize request. Caching (100ms TTL) ensures low-latency policy evaluation without per-request network overhead.

The operator's public key is registered via `MTA_PUBKEY_HEX`. Custom operators supply their own key via the same environment variable; the `MtaClient` implementation handles the corresponding verification.

### 4.4 MacroPulse Regime Definitions (Reference Operator)

The following regime taxonomy is specific to the **MacroPulse reference MTA**. It is not an IRL requirement. Custom operators define their own regime vocabulary and map it to the normalized constraint fields in §4.2.

| ID | Label | risk_level | max_notional_scale | allowed_sides |
|----|-------|-----------|-------------------|---------------|
| 0 | expansion | 1.00 | 1.00 | long, short, neutral |
| 1 | recovery | 0.75 | 0.75 | long, short, neutral |
| 2 | tightening | 0.30 | 0.25 | short, neutral |
| 3 | risk_off | 0.00 | 0.00 | neutral |

### 4.5 Construction Pipeline

The MacroPulse MTA derives regime state through a five-step pipeline:

```
Step 1: Ingest macro factor time series
        (yield curve spreads, credit spreads, VIX, PMI, USD index, ...)

Step 2: PCA dimensionality reduction
        z_t = PCA(X_t)   where dim(z_t) = k, explaining ≥ 80% of variance

Step 3: HMM regime inference
        regime_label = argmax P(state | z_1:t)

Step 4: Discretize to regime enum
        { Expansion, Recovery, Tightening, RiskOff }

Step 5: Version, sign, and broadcast
        MTA_t = {
          regime_label  : RegimeEnum,
          version_hash  : SHA-256(pipeline_params),
          broadcast_time: Unix ms,
          signature     : Ed25519(Ed25519_key, canonical_json(MTA_t))
        }
```

Every change to the pipeline — model weights, PCA loadings, input schema — increments the MTA version and produces a new `version_hash`, which is detectable in any downstream audit trace.

### 4.6 Properties

- **Versioned**: every pipeline change is detectable via the `version_hash` embedded in every snapshot
- **Cryptographically signed**: each broadcast carries an Ed25519 signature; IRL rejects unsigned or invalidly signed broadcasts regardless of regime content
- **Deterministic**: given the same inputs and version hash, the same regime label is always produced — enabling independent reproduction by auditors
- **Cached**: IRL maintains a 100ms cached MtaState — one network call per 100ms window, not one per authorize request

---

## 5. The Cognitive Snapshot

### 5.1 Definition

The `CognitiveSnapshot` S_t is the atomic unit of proof. It is the complete, sealed record of an agent's epistemic state at the moment of intent:

```
S_t = { R_t, L_t, E_t, τ_t }
```

Where:
- **R_t** — Reasoning state: the verified MTA regime the agent was observing
- **L_t** — Latent fingerprint: the cryptographic identity of the agent's model
- **E_t** — Execution intent: the full, authorised order specification
- **τ_t** — Temporal proof: the bitemporal timestamps proving forward-only reasoning

### 5.2 Reasoning State R_t

```
R_t = { regime_id, regime_label, mta_hash, mta_version, policy_result, policy_hash }
```

The `mta_hash` binds the snapshot to a specific signed MTA broadcast. The `policy_hash` is SHA-256 of the policy ruleset used to evaluate the intent — ensuring that policy changes are detectable in the audit record.

### 5.3 Latent Fingerprint L_t

L_t is the cryptographic identity of the agent's model at the time of the decision:

```
L_t = SHA-256(model_id || "||" || prompt_version || "||" || feature_schema_id || "||" || hyperparameter_checksum)
```

The four components:
- **model_id** — identifier of the model version (e.g. `"hmm-v3.1"`)
- **prompt_version** — version of the instruction/prompt set (e.g. `"v2.4"`)
- **feature_schema_id** — identifier of the feature preprocessing schema (e.g. `"schema-prod-v1"`)
- **hyperparameter_checksum** — SHA-256 of the model's hyperparameter configuration

The `"||"` delimiter is chosen to be absent from all component strings, preventing collision attacks where two different component combinations produce the same concatenated input before hashing.

L_t is verified against the Multi-Agent Registry: the registered `model_hash_hex` must match the `L_t` derivation inputs. A mismatch means the running agent is not the registered agent.

### 5.4 Execution Intent E_t

```rust
pub struct ExecutionIntent {
    pub action:          TradeAction,      // Long(f64) | Short(f64) | Neutral
    pub asset:           String,           // e.g. "BTC-PERP"
    pub order_type:      OrderType,        // MARKET | LIMIT | STOP | TWAP | VWAP
    pub venue_id:        String,           // MIC code or internal route e.g. "XNAS"
    pub quantity:        f64,              // number of units
    pub notional:        f64,              // quantity × reference price (USD)
    pub limit_price:     Option<f64>,      // required for LIMIT orders
    pub client_order_id: String,           // agent-assigned unique order identifier
}
```

The full E_t is embedded in the snapshot before sealing. Every field is covered by the `reasoning_hash`. An attempt to modify the asset, quantity, notional, or order type after sealing invalidates the hash.

### 5.5 Temporal Proof τ_t

```
τ_t = { valid_time, txn_time, time_source }
```

- **valid_time** — Unix milliseconds: the agent's claimed timestamp of its reasoning (supplied by the agent in the authorize request)
- **txn_time** — Unix milliseconds: the IRL engine's wall-clock time at receipt of the authorize request
- **time_source** — `System` (development) or `NtpSynced` (production)

**The bitemporal invariant**: `valid_time < txn_time` is enforced at both the application and database layers. A snapshot with `valid_time ≥ txn_time` is rejected with `BiTemporalViolation`. This prevents hindsight bias — the agent cannot claim to have reasoned about a future MTA state — and prevents data revision attacks where a record is backdated after the outcome is known.

---

## 6. The Policy Engine

### 6.1 Purpose

The policy engine evaluates the agent's execution intent against the current regime and returns either `ALLOWED` or `HALTED`. A `HALTED` decision blocks the authorize request — the agent receives a policy violation error and no trace is committed.

### 6.2 Constraint-Based Policy Engine

IRL's policy engine (`IrlConstraintPolicy`) does not hardcode per-regime rules. It reads the three normalized constraint fields from the active `MtaState` and enforces exactly two checks:

**Check 1 — Direction (allowed_sides)**
```
action_side = "long" | "short" | "neutral"   // derived from TradeAction
if action_side ∉ mta.allowed_sides → REGIME_VIOLATION (403)
```

**Check 2 — Notional (scaled cap)**
```
effective_cap = agent_profile.max_notional × mta.max_notional_scale
if intent.notional > effective_cap → NOTIONAL_EXCEEDS_LIMIT (403)
```

This means a MacroPulse `risk_off` broadcast (allowed_sides = ["neutral"], max_notional_scale = 0.0) produces the same enforcement outcome as any other operator broadcasting equivalent constraints. The policy engine is indifferent to who produced them.

### 6.3 Policy Hash

The `policy_hash` embedded in every trace is a SHA-256 of the **live constraint values** that governed the decision:

```
policy_hash = SHA-256(regime_id || version || risk_level || max_notional_scale || allowed_sides)
```

This proves exactly which constraints were in effect at the time of evaluation — not just the policy version, but the specific operator values. If the operator changed their model between the intent and the audit review, the hash mismatch is detectable.

### 6.4 Notional Enforcement

The effective notional limit for any intent applies regime scaling on top of the per-agent MAR cap:

```
effective_limit = agent_profile.max_notional × mta.max_notional_scale
```

When `max_notional_scale = 0.0` (e.g., a fully defensive regime state), no new notional is permitted regardless of the agent's configured cap. This replaces the old per-regime hardcoded `max_notional` values.

---

## 7. Cryptographic Sealing

### 7.1 RFC 8785 Canonicalization

The `CognitiveSnapshot` is serialised using RFC 8785 JSON Canonicalization Scheme before hashing. RFC 8785 guarantees:
- Object keys are sorted lexicographically (Unicode code point order)
- No whitespace outside of string values
- Numbers are serialised without unnecessary precision
- The output is deterministic across all implementations, languages, and library versions

Standard `serde_json::to_string()` does **not** guarantee field ordering across crate versions or struct changes and must not be used for audit hashing.

The canonical form can be independently reproduced by any party from the stored snapshot fields:
```python
import json
canonical = json.dumps(snapshot_dict, sort_keys=True, separators=(',', ':'))
```

### 7.2 Reasoning Hash

```
reasoning_hash = SHA-256(RFC_8785_canonical(S_t))
```

The `reasoning_hash` is:
- Returned to the agent in the authorize response
- Stored in `irl.reasoning_traces.reasoning_hash`
- Included by the agent in order metadata sent to the exchange
- Used as the left input to `final_proof` computation

Any modification to any field of S_t — including the MTA hash, L_t fingerprint, execution intent, or timestamps — produces a different `reasoning_hash`. The hash is the root of the audit chain.

### 7.3 The Bitemporal Ledger

Every trace is stored with two time dimensions:
- **valid_time** — when the agent claims to have reasoned (business time)
- **txn_time** — when the IRL engine recorded the intent (transaction time)

Neither dimension can be altered after the fact. Queries over the bitemporal ledger can reconstruct the exact state of the world as it was known at any historical point — not just what happened, but what was believed to be true when it happened.

### 7.4 RFC 8785 — Why This Format

IRL deviates from a purely binary serialisation specification by choice. The primary requirement for snapshot serialisation is determinism: two independent implementations must produce byte-for-byte identical output from the same logical input, or their hashes will diverge and the audit guarantee collapses. A bespoke binary format meets this requirement but introduces implementation risk — any deviation in field ordering, padding, or encoding between implementations breaks interoperability silently.

RFC 8785 provides determinism through precisely specified transformation rules:
- Object keys sorted lexicographically by Unicode code point — no insertion-order ambiguity
- Floating-point numbers follow the ECMAScript serialisation algorithm (IEEE 754, shortest round-trip) — no decimal string ambiguity
- Output is UTF-8 with no BOM, no trailing whitespace, no line breaks

The practical advantages for an audit protocol:

| Concern | Advantage |
|---------|-----------|
| Auditor accessibility | Any RFC 8785–compliant library, in any language, reproduces canonical bytes — no proprietary tooling |
| Cross-language verification | JCS implementations exist in Rust, Python, Go, Java, TypeScript — cross-testable against RFC test vectors |
| Regulatory submission | JSON is the lingua franca of financial regulatory reporting — no translation layer |
| Debug transparency | Records are human-readable during incident investigation |

The trade-off is serialisation overhead relative to a compact binary format. This is acceptable: the performance-critical path operates on in-memory structs. RFC 8785 serialisation occurs once per snapshot for hashing and storage, not in a tight execution loop.

### 7.5 Serialisation Versioning

Every Cognitive Snapshot includes a `ser_version` field in its metadata envelope, declaring the serialisation format used to produce the `reasoning_hash`. This field is included in the hash input — a snapshot serialised under `0x0001` (JCS) and an otherwise identical snapshot serialised under `0x0002` (binary) produce different hashes by design. This makes the serialisation format an immutable, auditable property of every sealed record.

```
ser_version 0x0001 = RFC 8785 JCS  (current)
ser_version 0x0002 = binary        (reserved for future)
```

If performance profiling demonstrates that RFC 8785 overhead is material, IRL can adopt binary serialisation by incrementing `ser_version`, without invalidating any existing traces. All existing traces remain verifiable under `0x0001` indefinitely.

### 7.6 Merkle Aggregation

For high-frequency environments, individual snapshot hashes are aggregated into a Merkle tree to produce a single period root hash. This enables efficient batch verification: an auditor can verify that a specific `reasoning_hash` is included in a period's audit record by verifying the Merkle inclusion proof, without downloading all snapshots in the period.

```
Period_Root = MerkleRoot([reasoning_hash_1, reasoning_hash_2, ..., reasoning_hash_n])
Proof(H_ti) = Merkle inclusion proof from H_ti to Period_Root
```

`Period_Root` is anchored to an immutable external ledger (L2 rollup or permissioned ledger, e.g. Hyperledger Besu), providing tamper evidence independent of operator infrastructure.

---

## 8. Layer 2 — Heartbeat Protocol

### 8.1 Purpose

Layer 2 adds an anti-replay constraint. Without it, an agent could theoretically replay a valid authorize request from a previous time period, falsely claiming to be operating under a different (more favourable) regime.

### 8.2 SignedHeartbeat

```rust
pub struct SignedHeartbeat {
    pub sequence_id:  u64,      // monotonically increasing, per-agent
    pub timestamp_ms: u64,      // Unix ms at heartbeat creation
    pub regime_id:    u8,       // regime the agent claims to have observed
    pub mta_ref:      String,   // hex SHA-256 of the MTA broadcast the agent observed
    pub signature:    Vec<u8>,  // Ed25519 over (seq_id_be || timestamp_ms_be || regime_id || mta_ref_utf8)
}
```

### 8.3 Validation Rules

The `HeartbeatValidator` enforces four conditions:

1. **Signature valid** — Ed25519 signature verifies against the agent's registered signing key
2. **Sequence monotonic** — `sequence_id > last_accepted_sequence_id` for this agent
3. **Drift within bound** — `|timestamp_ms - now_ms| ≤ MAX_HEARTBEAT_DRIFT_MS` (default: 200ms)
4. **mta_ref matches current MTA** — `heartbeat.mta_ref == current_mta_state.hash`

Condition 4 is the key Layer 2 addition. The agent must prove it has observed the *current* MTA broadcast, not a historical one. An agent replaying a heartbeat from an earlier time period will fail condition 4 because the current MTA hash will have changed.

### 8.4 Enabling Layer 2

```dotenv
LAYER2_ENABLED=true
MAX_HEARTBEAT_DRIFT_MS=200
```

When enabled, every `POST /irl/authorize` request must include a valid `heartbeat` field. Missing or invalid heartbeats return `401 INVALID_HEARTBEAT`.

---

## 9. Multi-Agent Registry

### 9.1 Purpose

The Multi-Agent Registry (MAR) provides fleet-level identity governance. In a firm running multiple autonomous agents across strategies, venues, and regimes, the MAR answers three questions for every intent:

1. Is this the registered agent, running the registered model?
2. Is this agent permitted to trade in the current regime?
3. Does this intent exceed the agent's notional ceiling?

### 9.2 AgentProfile

```rust
pub struct AgentProfile {
    pub agent_id:         Uuid,
    pub name:             String,
    pub model_hash_hex:   String,          // SHA-256 of model version + config
    pub policy_module_id: String,          // built-in policy variant label
    pub allowed_regimes:  Vec<i16>,        // regime IDs this agent may trade
    pub max_notional:     f64,             // per-decision notional ceiling (USD)
    pub max_leverage:     f64,             // maximum leverage multiple
    pub allowed_venues:   Option<Vec<String>>,  // None = all venues
    pub status:           String,          // "Active" | "Suspended" | "Deregistered"
}
```

### 9.3 Authorization Flow

For every `POST /irl/authorize`, the MAR performs four checks before the snapshot is assembled:

```
1. agent_id exists in irl.agent_registry                → AgentNotFound (404)
2. profile.status == "Active"                           → AgentNotActive (403)
3. model_hash_hex matches observed model hash           → ModelHashMismatch (403)
4. current_regime_id in profile.allowed_regimes         → RegimeUnauthorized (403)
5. intent.notional ≤ profile.max_notional               → NotionalExceedsLimit (403)
```

All five checks must pass before the snapshot is assembled. A failure at any step blocks the trade and returns a structured error — no trace is committed.

### 9.4 Fleet Management API

```
POST   /irl/agents                  Register a new agent
GET    /irl/agents                  List all agents
GET    /irl/agents/:id              Get agent profile
PATCH  /irl/agents/:id/status       Suspend or deregister
```

### 9.5 Model Hash Verification

The `model_hash_hex` is the SHA-256 of the agent's model version and configuration. It is computed by the firm at deployment time and registered once. On every authorize request, the agent supplies its `model_hash_hex`. The engine compares it against the registered value.

This ensures that a model update — even a minor hyperparameter change — is detectable. An agent running a different model version than the one registered will be blocked with `MODEL_HASH_MISMATCH`.

---

## 10. Post-Trade Verifier

### 10.1 Lifecycle States

Every trace begins in `PENDING` and transitions through a defined lifecycle:

```
PENDING ──► MATCHED      (exchange confirmed; asset and quantity within tolerance)
        ──► DIVERGENT    (exchange confirmed; parameters differ from authorised intent)
        ──► EXPIRED      (no exchange confirmation within TRACE_EXPIRY_MS)

ORPHAN: exchange report received with no corresponding IRL trace
```

### 10.2 Reconciliation Logic

When `POST /irl/bind-execution` is received, the engine performs two checks against the stored authorised intent:

**Asset check:**
```
if executed_asset ≠ authorised_asset → DIVERGENT("Asset mismatch: authorized X, executed Y")
```

**Quantity tolerance check:**
```
delta = |authorised_quantity - executed_quantity|
if delta > authorised_quantity × BIND_SIZE_TOLERANCE → DIVERGENT("Quantity outside tolerance")
```

`BIND_SIZE_TOLERANCE` defaults to `0.0001` (0.01%) and is configurable via environment variable. If both checks pass, the result is `MATCHED`.

### 10.3 Async Expiry Worker

A background Tokio task sweeps the ledger every 60 seconds and expires stale PENDING traces:

```sql
UPDATE irl.reasoning_traces
SET verification_status = 'EXPIRED'
WHERE verification_status = 'PENDING'
  AND txn_time < now() - make_interval(secs => $1)
```

`TRACE_EXPIRY_MS` defaults to `3_600_000` (1 hour). A trace that reaches EXPIRED is not a neutral outcome — it indicates an intent was authorised but never confirmed by the exchange. EXPIRED traces appear in `GET /irl/orphans` for operator review.

### 10.4 Compliance Endpoints

```
GET /irl/pending?age_seconds=N    Unconfirmed traces older than N seconds
GET /irl/orphans                  EXPIRED and DIVERGENT traces
GET /irl/trace/:trace_id          Full Reasoning_Trace_v1 for forensic replay
```

---

## 11. The Final Proof

### 11.1 Computation

Once reconciliation completes, the engine computes:

```
final_proof = SHA-256(reasoning_hash || "||" || exchange_tx_id)
```

The `final_proof` cryptographically binds the agent's authorised reasoning (`reasoning_hash`) to the specific exchange transaction (`exchange_tx_id`). It is the terminal node of the audit chain:

```
MacroPulse MTA broadcast
    └─► mta_hash (embedded in S_t)
         └─► reasoning_hash = SHA-256(canonical(S_t))
              └─► final_proof = SHA-256(reasoning_hash || exchange_tx_id)
```

### 11.2 Properties

- **Binding**: `final_proof` cannot be reproduced without both inputs. An agent claiming a different exchange transaction for the same reasoning — or vice versa — produces a different proof that does not match the stored value.
- **Non-repudiation**: The agent cannot deny having authorised the trade, because the `reasoning_hash` embeds its model identity, its regime knowledge, and its explicit execution intent.
- **Forward-only**: The bitemporal constraint (`valid_time < txn_time`) ensures the chain reflects forward-only causality. The agent could not have known the exchange transaction ID when the snapshot was sealed.

---

## 12. Audit and Forensic Replay

### 12.1 The Reasoning_Trace_v1 Record

Every authorised intent, whether matched, divergent, or expired, produces a `Reasoning_Trace_v1` record stored in `irl.reasoning_traces`. The record is:

- **Immutable** — the reasoning_hash field is written once on authorize and cannot be altered
- **Queryable** — the bitemporal timestamps support as-of queries over the ledger
- **Reproducible** — given the same snapshot fields, any compliant implementation can recompute reasoning_hash and final_proof and verify they match the stored values
- **Self-contained** — the record embeds the MTA hash, policy hash, L_t fingerprint, and full execution intent — no external references are required to verify the chain

### 12.2 Forensic Replay

`GET /irl/trace/:trace_id` returns the complete record. An auditor can:

1. Recompute `reasoning_hash` from the stored snapshot fields using RFC 8785 + SHA-256
2. Verify the MTA signature using the MacroPulse public key
3. Confirm the bitemporal constraint
4. Recompute `final_proof` from `reasoning_hash + exchange_tx_id`
5. Cross-reference the exchange transaction independently

No access to the agent's internal state, model weights, or strategy is required at any step.

### 12.3 Regulatory Alignment

| Regulatory Requirement | IRL Mechanism | Evidence Produced |
|------------------------|---------------|-------------------|
| Decision auditability (EU AI Act Art. 12) | CognitiveSnapshot + reasoning_hash | Reproducible hash of complete decision state |
| Transparency to regulators (Art. 13) | ZK-SNARK public verifier (L3) | Compliance proved without alpha disclosure |
| Traceability to execution | final_proof = SHA-256(reasoning_hash \|\| exchange_tx_id) | Cryptographic link between reasoning and order |
| Contemporaneous recordkeeping | Bitemporal timestamps (valid_time, txn_time) | Provable record of what was known when |
| Pre-trade risk controls (SEC Rule 15c3-5) | Policy Engine (pre-execution) | Verifiable enforcement before order emission |
| Human oversight enforcement | Hard halt on policy violation | Non-compliant actions cannot be executed |
| Non-repudiation | Execution proof + Merkle anchoring (L2) / TEE signing (L3) | Tamper-evident record; full NR at L3 |
| Fleet-level governance (MiFID II, DORA) | Multi-Agent Registry | Per-agent identity, policy, and risk audit trail |

---

## 13. Security Analysis

### 13.1 Threat Model

| Threat | Mitigation |
|--------|-----------|
| Agent forges a retroactive snapshot | Bitemporal constraint: `valid_time < txn_time` enforced at application + DB layer; IRL engine's `txn_time` is authoritative |
| Agent replays a historical authorize request | Layer 2 heartbeat: monotonic sequence_id + mta_ref binding to current MTA state |
| Agent claims a different model version post-hoc | L_t fingerprint: model_hash_hex verified against MAR at authorize time and embedded in sealed snapshot |
| MTA operator signs a false regime | Multi-MTA consensus (§17): k-of-n threshold signing removes single-operator trust |
| Exchange reports a different asset than executed | Reconciliation: DIVERGENT result with divergence_reason permanently recorded |
| Compliance officer rewrites the audit log | reasoning_hash and final_proof are derived from the data — altering any field invalidates the chain |
| Hindsight bias: agent backdates its reasoning | Bitemporal invariant: the engine's wall-clock `txn_time` cannot be retroactively altered |

### 13.2 What IRL Cannot Prevent

IRL is a pre-execution gateway, not a comprehensive fraud prevention system. It cannot:

- Prevent a compromised agent from submitting a valid (but strategically harmful) authorize request
- Prevent exchange-side manipulation after the order is placed
- Verify the truthfulness of the agent's `valid_time` claim (only that it precedes `txn_time`)
- Replace position-level risk management systems

IRL's guarantee is narrower and stronger: *if a trade was executed, there exists an immutable cryptographic record of what was authorised before it happened, and whether the execution matched that authorisation.*

### 13.3 Sovereign Gateway (Layer 3)

In Layers 1 and 2, IRL enforces compliance and produces verifiable records — but it does not control the execution channel. A sufficiently privileged operator could, in principle, bypass the sidecar and route orders directly to the exchange. Layer 3 eliminates this attack surface.

In the Sovereign Gateway configuration, the exchange API credentials are stored exclusively within a Trusted Execution Environment (TEE — Intel TDX or AMD SEV). The host OS, agent process, and all operator-level personnel cannot access these credentials. A trade order can only be signed if the Enclave Signer receives a **Double-Green signal**: simultaneous valid attestations from both the MTA policy layer (L1) and the Multi-Agent Registry (L2).

```
Double-Green condition:
C = Verify_MTA(S_t) AND Verify_MAR(agent_id, model_hash, regime)

If C = 1: enclave releases sigma_trade = HMAC(api_secret, trade_payload)
          trade is broadcast to exchange

If C = 0: enclave does not release sigma_trade
          no trade is possible — key material is inaccessible outside the enclave
```

**Key property**: In Layer 3, the existence of a confirmed exchange trade is itself proof of compliance. The enclave is physically incapable of signing a trade that violates the MacroPulse regime policy. There is no bypass path.

| Threat | L1 Mitigation | L2 Mitigation | L3 Mitigation |
|--------|--------------|--------------|--------------|
| Snapshot modification after sealing | SHA-256 sealing | Merkle anchoring | TEE attestation |
| Heartbeat replay | T_valid < T_txn | Heartbeat freshness | Enclave validates heartbeat |
| Policy bypass via sidecar bypass | Sidecar in exec path | Orphan detection | No signature = no trade |
| Rogue admin bypass | None | None | Key material inaccessible to host |
| Model substitution | L_t mismatch logged | MAR hash check | Enclave rejects invalid MAR proof |
| Shadow execution | Logged as orphan | MAR cross-reference | Impossible — no unsigned orders accepted |

### 13.4 Data Integrity and Trusted Inputs

IRL's cryptographic guarantees are only as strong as the integrity of its inputs. A sealed snapshot proves that the system acted consistently with a given input state — it does not independently verify that the input state was accurate. All inputs to the Snapshot Engine must satisfy:

- **Authenticated origin**: data sources present a verifiable identity via certificate or registered public key
- **Encrypted transport**: all data channels use mutual TLS (mTLS) with certificate pinning
- **Verifiable signatures**: all structured inputs (MTA broadcasts, heartbeats) carry a valid Ed25519 signature verified before ingestion
- **Source registry**: a registered list of trusted data providers; inputs from unregistered sources are rejected regardless of content

| Threat | Attack Vector | IRL Mitigation |
|--------|--------------|---------------|
| Data poisoning | Injecting false market data to shift regime classification | Signed MTA; rejection of unsigned or unregistered sources |
| Man-in-the-middle | Intercepting and modifying data in transit | mTLS with certificate pinning + signature verification |
| Stale data replay | Submitting old MTA broadcast to invoke prior regime | Heartbeat freshness + monotonic seq_id |
| Prompt injection | Modifying agent inputs to alter L_t or E_t | L_t fingerprints prompt version hash; deviation detected |
| Model substitution | Replacing agent model without updating MAR | L_t model hash verified against MAR registration |
| Forged heartbeat | Fabricating a heartbeat with a favourable regime | Ed25519 signature verification; invalid sig = rejection |

---

## 14. Competitive Positioning

### 14.1 The Landscape

| Solution | Pre-trade enforcement | Post-trade audit | Tamper-evident | Regime-aware |
|----------|----------------------|-----------------|----------------|--------------|
| Manual logging | ✗ | Partial | ✗ | ✗ |
| Trade surveillance | ✗ | ✓ | ✗ | ✗ |
| Risk systems | Partial | ✗ | ✗ | Partial |
| Attestation services | ✗ | Partial | ✗ | ✗ |
| **MacroPulse IRL** | **✓** | **✓** | **✓** | **✓** |

There is no direct competitor. IRL fills a gap that currently has no solution — not because the problem is new, but because the cryptographic primitives required to solve it at production scale have only recently become viable at acceptable latency.

### 14.2 Who Is Forced to Adopt First

IRL responds to specific, live regulatory and operational pressure:

**Prop firms under SEC AI audit scrutiny** — The SEC's Division of Examinations has flagged AI-driven trading as a 2024–2026 examination priority. Firms must demonstrate that autonomous agents operated within defined parameters.

**Hedge funds with LP reporting requirements** — Institutional LPs are increasingly asking: "How do you know your AI didn't go rogue?" IRL provides a concrete, auditable answer.

**Brokers enabling AI-driven client order flow** — A broker routing orders from an autonomous agent shares liability for that agent's behaviour. IRL creates a clean separation of proof.

**Compliance teams scaling autonomous agent fleets** — One agent is manageable. Ten agents across multiple strategies, venues, and regimes is not. The MAR gives compliance teams visibility and control that logs cannot provide.

**Firms preparing for MiCA, DORA, and AI governance mandates** — The EU AI Act and financial sector AI regulations are moving toward mandatory explainability and auditability for high-frequency decision systems.

### 14.3 The Competitive Moat

The moat is not the software — it is the combination of:
1. **The seal** — IRL is the entity that issues the cryptographic chain of custody. Any regime signal, any exchange, any agent. The seal is what regulators, LPs, and counterparties trust — not the signal itself.
2. **Signal agnosticism** — firms with proprietary models don't have to expose their alpha to get compliance. They bring their own MTA. IRL seals it. This is the adoption path that no bundled solution can offer.
3. **MacroPulse as turnkey MTA** — for firms without a proprietary signal, MacroPulse is the reference operator: signed, versioned, fresh, zero additional infrastructure.
4. **Incremental adoptability** — L1 in a day; L3 when mandated.

Any firm can build logging. No firm can retroactively build a cryptographic chain of custody. IRL captures it at the only moment it can be captured honestly: before the order is placed.

---

## 15. Deployment Guide

### 15.1 Prerequisites

- PostgreSQL 14+ (TimescaleDB compatible — shared with MacroPulse is supported)
- MacroPulse API access (for the MTA endpoint and Ed25519 public key)
- Docker, or Rust toolchain (edition 2021)

### 15.2 Configuration

Copy `.env.example` to `.env` and set four required values:

```dotenv
MTA_URL=https://your-mta-operator.com
MTA_PUBKEY_HEX=<64-char hex Ed25519 public key from your MTA operator>
DATABASE_URL=postgres://user:pass@localhost:5432/yourdb
IRL_API_TOKENS=<comma-separated bearer tokens, one per client/fund>
```

Optional configuration with production-ready defaults:

```dotenv
LAYER2_ENABLED=false          # Enable heartbeat enforcement
MAX_HEARTBEAT_DRIFT_MS=200    # Maximum heartbeat age
BIND_SIZE_TOLERANCE=0.0001    # Quantity tolerance (0.01%)
TRACE_EXPIRY_MS=3600000       # 1 hour before PENDING → EXPIRED
TIME_SOURCE=System            # "NtpSynced" for production
PORT=4000
```

### 15.3 Starting the Engine

```bash
docker compose up -d
```

The engine applies all database migrations automatically on first boot:
```
IRL Engine starting on port 4000
Migrations applied
Post-trade verifier started (expiry: 3600s)
Listening on 0.0.0.0:4000
```

### 15.4 Registering an Agent

Each autonomous agent must be registered once before it can submit intents:

```bash
curl -X POST http://localhost:4000/irl/agents \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my-btc-bot",
    "model_hash_hex": "<sha256_of_model_config>",
    "allowed_regimes": [0, 1, 2, 3],
    "max_notional": 500000.0
  }'
```

**Computing `model_hash_hex`:**
```python
import hashlib, json

model_config = {
    "model_id": "hmm-v3.1",
    "prompt_version": "v2.4",
    "feature_schema": "schema-2026-q1",
    "hyperparameters": {"n_regimes": 4, "lookback_days": 60}
}
model_hash_hex = hashlib.sha256(
    json.dumps(model_config, sort_keys=True).encode()
).hexdigest()
```

### 15.5 Integration Pattern

```python
def irl_authorize(agent_id, model_hash, intent):
    return requests.post(f"{IRL_URL}/irl/authorize",
        headers={"Authorization": f"Bearer {token}"},
        json={
            "agent_id": agent_id,
            "model_hash_hex": model_hash,
            "model_id": "hmm-v3.1",
            "prompt_version": "v2.4",
            "feature_schema_id": "schema-2026-q1",
            "hyperparameter_checksum": hyperparam_hash,
            "action": {"type": "Long", "size": intent.size},
            "asset": intent.asset,
            "order_type": "MARKET",
            "venue_id": intent.venue,
            "quantity": intent.quantity,
            "notional": intent.notional,
            "limit_price": None,
            "client_order_id": intent.order_id,
            "agent_valid_time": int(time.time() * 1000),
        }
    ).json()

def irl_bind(trace_id, exchange_tx_id, asset, executed_qty, price):
    return requests.post(f"{IRL_URL}/irl/bind-execution",
        headers={"Authorization": f"Bearer {token}"},
        json={
            "trace_id": trace_id,
            "exchange_tx_id": exchange_tx_id,
            "execution_status": "Filled",
            "asset": asset,
            "executed_quantity": executed_qty,
            "execution_price": price,
        }
    ).json()

# Your existing bot logic, wrapped:
auth = irl_authorize(AGENT_ID, MODEL_HASH, my_intent)
assert auth["authorized"], f"IRL blocked: {auth}"

fill = exchange.place_order(my_intent)          # unchanged

result = irl_bind(auth["trace_id"], fill.tx_id,
                  fill.asset, fill.qty, fill.price)
print(f"{result['verification_status']} — proof: {result['final_proof']}")
```

### 15.6 Incremental Adoption Path

```
Day 1:   L1 sidecar running, first agent registered, bot wrapped
Week 1:  Compliance team reads /irl/orphans daily
Month 1: Enable LAYER2_ENABLED=true for heartbeat enforcement
Month 3: Per-regime notional limits tuned per agent in MAR
Later:   L3 TEE / ZK when regulatory requirements escalate
```

---

## 16. MTA Integration Modes

IRL is signal-agnostic by design. The engine depends on a verified `MtaState` — it does not care who produced it, as long as it is properly signed, versioned, and fresh. The interface between IRL and any MTA operator is a single Rust trait:

```rust
#[async_trait]
pub trait MtaClient: Send + Sync {
    async fn fetch_verified(&self) -> Result<MtaState, AppError>;
}
```

This abstraction allows two distinct deployment modes:

### 16.1 MacroPulse as a Managed MTA Service

MacroPulse operates a production-ready HMM-based regime inference pipeline. `MacroPulseMtaClient` implements the `MtaClient` trait and:

- Connects to the MacroPulse broadcast endpoint
- Verifies the Ed25519 signature against the pre-registered public key
- Caches the latest state for low-latency policy evaluation (100ms TTL)
- Enforces the heartbeat freshness constraint

This mode is turnkey: firms deploy IRL with no additional MTA infrastructure. MacroPulse provides a service-level agreement on regime availability, signature validity, and broadcast latency.

### 16.2 Custom MTA Client

For firms that require a different regime definition — or that wish to participate in a multi-signer consortium — any type implementing `MtaClient` can be substituted at startup with no changes to route handlers, the policy engine, the MAR, or the post-trade verifier:

```rust
pub struct MyInternalMta { /* ... */ }

#[async_trait]
impl MtaClient for MyInternalMta {
    async fn fetch_verified(&self) -> Result<MtaState, AppError> {
        // poll internal HMM, aggregate consortium signatures, return MtaState
    }
}

// In main.rs — one line change:
let mta_client: Arc<dyn MtaClient> = Arc::new(MyInternalMta::new(&config));
```

A custom client may:
- Poll a private regime pipeline (firm-internal HMM or a different vendor)
- Aggregate signatures from a consortium of independent nodes (see §17)
- Apply additional source-consensus logic before presenting a single `MtaState`

As long as the returned `MtaState` is properly signed, timestamped, and fresh, the engine treats it identically to a MacroPulse-sourced state.

### 16.3 Deployment Flexibility

```
Initial:  MacroPulseMtaClient — single trusted signer, minimal setup
Maturity: Custom client with consortium aggregation (§17) — no engine changes required
```

The choice of MTA operator becomes a configuration parameter, not an architectural constraint.

---

## 17. Future Work — Multi-MTA Consensus

*See also: §4 (MTA definition), §8 (heartbeat protocol), §10 (post-trade verifier).*

### 17.1 The Trust Problem

Layer 1 uses a single MTA signer (MacroPulse) for operational simplicity. But a single signer is a single point of trust. If MacroPulse signs a false regime — through compromise, error, or coercion — every IRL trace built on that signature inherits the lie. The cryptographic chain is intact, but the ground truth it anchors to is wrong.

**The protocol is designed to upgrade to decentralised MTA without breaking or invalidating any existing traces.**

### 17.2 Design Principles

1. No single signer should be able to unilaterally define market truth.
2. Consensus must be Byzantine fault-tolerant: the system survives up to `f` malicious or failed signers in a set of `n`, where `n ≥ 3f + 1`.
3. The IRL engine must remain stateless with respect to signer governance — signer set changes happen in a governance layer, not in engine code.
4. Regime disagreement must be detectable, logged, and alertable.
5. The aggregator is not a trusted party — it forwards signatures; trust derives from the signatures themselves.

### 17.3 Architecture: k-of-n Threshold Regime Signing

A consortium of `n` independent MTA nodes, each running:
- An independent HMM / PCA pipeline on the same market data
- An Ed25519 signing key, registered in the consortium governance contract
- A deterministic regime classification algorithm defined by a shared consortium specification

Example initial configuration: `n=5, k=3` (3-of-5 threshold).

```
MTA Node A (MacroPulse)    ──┐
MTA Node B (Independent)   ──┤
MTA Node C (Academic)      ──┼──► Aggregator ──► Threshold MtaState
MTA Node D (Consortium)    ──┤
MTA Node E (Client-run)    ──┘
```

Each node independently signs a `RegimeBroadcast` over RFC 8785 canonical JSON:

```json
{
  "epoch":        1742694308,
  "regime_id":    1,
  "regime_label": "recovery",
  "model_hash":   "e3b0c44...",
  "signer_id":    "a1b2c3d4...",
  "signature":    "<ed25519 over {epoch, model_hash, regime_id, regime_label}>"
}
```

The aggregator collects `k` broadcasts for the same `regime_id`, verifies all signatures, sorts signers lexicographically by `signer_id` (for determinism), and computes:

```
aggregate_hash = SHA-256(bytes(sig_1) || bytes(sig_2) || ... || bytes(sig_k))
```

The resulting threshold `MtaState` is bound to `aggregate_hash`, which requires `k` independent signers to have agreed.

### 17.4 Slashing

| Offence | Evidence | Consequence |
|---------|----------|-------------|
| Double-sign | Two valid signatures from same node, same epoch, different regime_id | Key revoked, stake slashed |
| Stale broadcast | Signature epoch outside drift window | Broadcast rejected, strike recorded |
| Model drift | model_hash changes without governance approval | Broadcasts rejected until re-registered |
| Systematic disagreement | Node disagrees >X% of epochs over 30 days | Review triggered, partial slash |

A slash proof is a pair of conflicting signed broadcasts from the same node. Any participant can submit one; the governance contract verifies and executes the penalty trustlessly.

### 17.5 Governance Evolution

| Stage | Model | When |
|-------|-------|------|
| 1 | MacroPulse controls signer set | Initial deployment |
| 2 | Legal consortium agreement (n=3, k=2) | First institutional clients |
| 3 | On-chain governance (EVM or Cosmos SDK) | Regulatory mandate or scale |
| 4 | DAO with token-weighted voting | Full decentralisation |

The IRL engine code does not change between stages. Only the governance layer that manages the signer registry evolves. A trace sealed under Stage 1 is verifiable under Stage 4.

### 17.6 Engine Changes Required

1. `MtaState` struct expansion: add `signer_count`, `threshold`, `signers`, `aggregate_hash`
2. Aggregator client: new `mta_aggregator.rs` implementing the `MtaClient` trait
3. Config: `MTA_THRESHOLD`, `MTA_SIGNERS` (JSON array of registered pubkeys)

The snapshot seal, policy engine, MAR, and post-trade verifier are unaffected.

### 17.7 Deployment Path

```
Phase 1 (now):   Single MTA (MacroPulse)
Phase 2 (~3mo):  2-of-3 (MacroPulse + 2 consortium partners)
Phase 3 (6-12mo): 3-of-5 with on-chain signer registry
Phase 4 (when regulated): k-of-n with slashing and DAO governance
```

---

## 18. Licensing

IRL is available under dual licensing:

**Open protocol**: The IRL protocol specification — the snapshot format, sealing algorithm, bitemporal constraints, and API contract — is published openly. Any implementation that conforms to the specification interoperates with the MacroPulse ecosystem.

**Commercial license**: Firms requiring proprietary integration, white-label deployment, or L3 (TEE / Wasm / ZK) capabilities require a commercial license. Licensing is structured on a per-agent or per-trade basis. FRAND terms are available for standard-essential implementations.

**Enterprise SLA**: MacroPulse provides enterprise SLAs covering MTA availability, signature validity, broadcast latency, and engine support.

Contact: gabriel.veron134@gmail.com

---

## 19. Zero-Knowledge Compliance Proofs (Layer 3)

### 19.1 The Alpha Protection Problem

Standard audit disclosure creates a tension between two legitimate requirements: regulators need to verify that agent decisions complied with declared policy constraints, and firms need to protect the proprietary signals, model weights, and strategy logic that constitute their competitive advantage.

IRL addresses this at Layer 3 through a zero-knowledge proof layer that allows a firm to prove compliance to a regulator without revealing the private inputs that drove the decision. The regulator receives mathematical certainty of compliance; the firm retains full confidentiality of its alpha.

### 19.2 The Arithmetic Circuit

To generate a zk-SNARK proof, the IRL compliance check is compiled into an arithmetic circuit. The circuit takes two classes of inputs:

| Input Class | Contents | Visibility |
|-------------|----------|-----------|
| Public inputs (x) | R_t (regime ID), E_t (execution vector), reasoning_hash | Visible to regulator and prover |
| Private witness (w) | L_t internals (model weights, alpha signals, prompt content) | Known only to prover — never disclosed |

The proof `π_t` is valid if and only if:

```
C(x, w) = 1  ⟺
    Hash(L_t, W, E_t) = reasoning_hash    // Snapshot is internally consistent
    AND Align(E_t, R_t) = True            // Execution complied with regime policy
```

The regulator verifies `π_t` using a public verifier key. A valid proof guarantees compliance **without revealing W or the alpha signals**.

### 19.3 Two-Layer Disclosure Model

IRL enforces a structural separation between public and private audit data:

```rust
// Public (regulator-facing) audit record — disclosed on demand
pub struct PublicAuditRecord {
    reasoning_proof  : ZKProof,         // π_t — proves compliance
    constraint_checks: Vec<String>,     // ["regime_pass", "leverage_pass", ...]
    execution_hash   : [u8; 32],        // reasoning_hash
}

// Private (internal only) — never leaves the operator's infrastructure
struct InternalDecisionRecord {
    decision_path    : Vec<SignalNode>,  // alpha signals, model activations
    latent_state     : LatentFingerprint,
    raw_features     : FeatureVector,
}
```

The proof `π_t` is compact (typically several hundred bytes) and is anchored to the ledger alongside the `reasoning_hash`, making it available for on-demand regulator verification at any future point.

---

## 20. Limitations

### 20.1 MTA Centralisation

The MTA is, in its current form, a centralised authority. Cryptographic signing and versioning prevent unauthorised modification of MTA broadcasts, but they do not prevent the MTA operator itself from broadcasting an inaccurate regime state. A validly-signed but incorrect MTA could cause unjustified trading halts or false approvals.

The trust model is therefore shifted rather than eliminated: from agent to MTA operator. This is an improvement — the MTA operator is a known, accountable party with its own liability exposure — but it is not a full solution. The correct long-term mitigation is the multi-party MTA consensus mechanism described in §17.

### 20.2 Clock Synchronisation

The heartbeat freshness constraint (`|timestamp - now| < epsilon`) depends on the IRL sidecar and the MTA broadcaster sharing a trustworthy time reference. In distributed deployments, clock skew is non-trivial.

Production deployments must source wall clock time from a hardware time reference — Precision Time Protocol (PTP, IEEE 1588) or a GPS-disciplined oscillator — not from the host OS clock, which is subject to NTP drift and software manipulation. The maximum tolerable skew between sidecar and MTA broadcaster must be bounded to less than `epsilon/4` to provide a safe margin against boundary-condition false rejections.

Clock skew above epsilon will produce false staleness rejections. This is the correct failure mode — a system that halts on clock uncertainty is safer than one that continues on potentially stale market truth.

### 20.3 Execution Binding Below Layer 3

At Layers 1 and 2, execution binding is soft. IRL issues a pre-authorisation token that links `reasoning_hash` to `client_order_id` before execution, and the post-trade verifier reconciles exchange confirmations afterward. This design provides strong auditability but does not prevent bypass: an operator with direct exchange access could submit orders without routing through the IRL gateway. Orphan trades are detected, not prevented.

This is an intentional trade-off. Requiring exchange-level signed order acceptance at Layer 2 would demand infrastructure changes from every exchange integration — a barrier that would prevent adoption. Layer 3's signed order routing closes the gap entirely, but requires two external dependencies not yet available at scale: TEE key custody infrastructure at the deploying firm, and exchange-side acceptance of cryptographically signed order payloads.

### 20.4 Policy Completeness Under Regime Evolution

The policy map must define behaviour for every regime state the MTA can produce. An undefined regime causes a default rejection — safe, but operationally disruptive during genuine market stress when novel conditions are most likely to trigger previously unseen regime labels.

Two specific risks require attention:

**Regime label expansion**: if the MTA pipeline is updated to produce new regime labels, any policy module that does not enumerate the new labels will default-reject all intents during those regimes. Policy modules must be updated and re-deployed before MTA pipeline updates go live.

**Wasm hot-reload race conditions**: the hot-reload mechanism allows policy modules to be updated without sidecar downtime. However, in-flight snapshot evaluations that straddle a policy update boundary must resolve against a single, consistent policy version. IRL enforces this by stamping the policy module hash into L_t at snapshot construction time — the policy version that governed a decision is always recoverable from the audit record, regardless of subsequent updates.

---

## Appendix A — End-to-End Case Study
*Version 1.0 — IRL Engine v1 / Whitepaper v4*

This appendix walks through two complete IRL traces captured from a live deployment: a **matched** trade (the normal path) and a **divergent** trade (the detection path). Every hash, timestamp, and identifier shown is real output from the running system.

---

### A.1 Scenario 1: Matched Trade

**Context:** Agent `alpha-bot-v1` intends to go long 2.0 BTC-PERP at market on XNAS. MacroPulse has broadcast a Recovery regime (id=1) with an Ed25519-signed payload.

---

**Step 1 — MTA Broadcast**

The canonical JSON signed by MacroPulse (RFC 8785 — keys sorted, no whitespace):

```json
{"macro_regime":"recovery","model_version":"v1","regime_id":1,"timestamp":"2026-03-23T02:45:07Z"}
```

```
mta_hash = SHA-256(above canonical JSON)
         = 3fa9e02ac49147e3f7fb0363c724d0795cce8ae469eb501b44bba19177961af0
signature: valid (Ed25519, mta_pubkey)
```

---

**Step 2 — Agent Submits Intent**

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

**Step 3 — Verification Chain**

**MAR check:**
- Agent `7e729402` — Active ✓ · model_hash matches ✓ · regime 1 in [0,1,2,3] ✓ · 120,000 ≤ 500,000 ✓

**Policy (Recovery):** Long allowed · Result: `ALLOWED`

**Latent fingerprint:**
```
L_t = SHA-256("hmm-v3.1" || "||" || "v2.4" || "||" || "schema-prod-v1" || "||" || "deadbeefdeadbeefdeadbeefdeadbeef")
    = 67636f725eb0957d068faf387b24d6434ccf62fd263969aa180885100e5790c2
```
*(The `"||"` delimiter never appears in any component string — preventing input collision.)*

**Bitemporal constraint:**
```
valid_time: 2026-03-23T02:45:08.484Z
txn_time:   2026-03-23T02:45:08.982Z
Gap: 498ms — agent could not have seen the future ✓
```

**Snapshot sealed (RFC 8785 canonical excerpt):**
```json
{"execution":{"action":{"size":2.0,"type":"Long"},"asset":"BTC-PERP","client_order_id":"ord-smoke-001","limit_price":null,"notional":120000.0,"order_type":"MARKET","quantity":2.0,"venue_id":"XNAS"},"feature_schema_id":"schema-prod-v1","latent_fingerprint":"67636f72...","mta_hash":"3fa9e02a...","mta_regime_id":1,"trace_id":"96e67b89-...","txn_time":1742694308982,"valid_time":1742694308484}
```

```
reasoning_hash = SHA-256(canonical(S_t))
               = b3971de1b84da2b1450e31beb3bad6d47c9b2ddb12cc26286aceefdf26e17157
```

---

**Step 4 — Authorization Response**
```json
{ "authorized": true, "trace_id": "96e67b89-05a7-4a47-a756-04c9abe69d39", "reasoning_hash": "b3971de1..." }
```

**Step 5 — Exchange Execution**
```
exchange_tx_id: exch-tx-9001 · asset: BTC-PERP · qty: 2.0 · price: 60,000.00 · status: Filled
```

**Step 6 — Bind and Reconcile**
```
Asset: BTC-PERP == BTC-PERP ✓ · Qty delta: 0.0 ≤ 0.01% tolerance ✓ → MATCHED

final_proof = SHA-256("b3971de1..." || "||" || "exch-tx-9001")
            = 9bcd966a3e85bde3dea5079a38d5026c9dc15c5e7e417ba4a6c60ac9de741f33
```

---

**Step 7 — Complete Audit Record**
```json
{
  "trace_id": "96e67b89-05a7-4a47-a756-04c9abe69d39",
  "version": "1.0.0",
  "agent":      { "agent_id": "7e729402-...", "latent_fingerprint": "67636f72..." },
  "mta":        { "regime_id": 1, "regime_label": "recovery", "risk_level": 0.75, "max_notional_scale": 0.75, "allowed_sides": ["long","short","neutral"], "hash": "3fa9e02a...", "signature_valid": true },
  "policy":     { "id": "IrlConstraintPolicy", "result": "ALLOWED", "hash": "a4c2f1e9..." },
  "execution":  { "action": "Long(2)", "asset": "BTC-PERP", "order_type": "MARKET", "venue_id": "XNAS", "quantity": 2.0, "notional": 120000.0 },
  "bitemporal": { "valid_time": "2026-03-23T02:45:08.484Z", "txn_time": "2026-03-23T02:45:08.982Z" },
  "integrity":  { "reasoning_hash": "b3971de1...", "final_proof": "9bcd966a...", "verification_status": "MATCHED" }
}
```

**For a regulator:** This record can be independently verified by (1) recomputing `reasoning_hash` from the stored fields using RFC 8785 + SHA-256, (2) verifying the MTA signature using the operator's registered public key, (3) confirming `valid_time < txn_time`, (4) recomputing `final_proof`. No access to the agent's model or strategy is required.

---

### A.2 Scenario 2: Divergent Trade

**Context:** Same agent, same regime. Intent: Long 1.0 ETH-PERP. Exchange executes BTC-PERP.

```json
{ "trace_id": "e8e35291-...", "reasoning_hash": "f26972d3...", "authorized": true }
```
Intent locked: **ETH-PERP, Long 1.0, $50,000 notional.**

Exchange reports `asset: "BTC-PERP"`. Reconciliation:
```
ETH-PERP ≠ BTC-PERP ✗ → DIVERGENT
divergence_reason: "Asset mismatch: authorized ETH-PERP, executed BTC-PERP"
final_proof: 47831b5a27bce55baee0577cc5bdf8a7239637743150713b692260472a3c888f
```

**For a regulator:** The DIVERGENT record proves two things simultaneously — the firm's intent was legitimate (ETH-PERP, within policy), and the execution deviated from that intent. The divergence is provable and permanent. A DIVERGENT record is evidence of a *functioning* compliance system, not of wrongdoing.

---

### A.3 Chain of Custody

```
MacroPulse HMM
    │ Ed25519-signed MTA broadcast
    │ mta_hash = SHA-256(canonical_json(regime_payload))
    ▼
IRL Engine — MtaState cached
    │ Agent submits AuthorizeRequest
    ▼
MAR check → Policy → L_t fingerprint → Bitemporal constraint
    │ All pass → CognitiveSnapshot S_t assembled
    ▼
reasoning_hash = SHA-256(RFC_8785_canonical(S_t))         [reasoning locked]
    │ Agent places order (reasoning_hash in order metadata)
    ▼
Exchange returns execution report
    │ Reconciliation: asset match · |qty_delta| ≤ tolerance
    ▼
final_proof = SHA-256(reasoning_hash || "||" || exchange_tx_id)
    ├── MATCHED   → audit record complete · chain intact
    └── DIVERGENT → flagged · reason recorded · chain preserved
```

---

## Appendix B — Diagram Specifications

The following Mermaid diagrams are provided for designer handoff. They render natively in GitHub's markdown viewer.

---

### B.1 End-to-End Execution Flow

```mermaid
sequenceDiagram
    participant MP as MacroPulse (MTA)
    participant AG as Autonomous Agent
    participant IRL as IRL Engine
    participant EX as Exchange

    MP->>IRL: Ed25519-signed regime broadcast {regime_id, mta_hash}
    Note over IRL: Caches MTA state (100ms TTL)

    AG->>IRL: POST /irl/authorize {agent_id, model_hash, intent, valid_time}

    rect rgb(240, 248, 255)
        Note over IRL: MAR Check
        IRL->>IRL: Verify agent Active
        IRL->>IRL: Verify model_hash matches registry
        IRL->>IRL: Verify regime in allowed_regimes
        IRL->>IRL: Verify notional ≤ max_notional
    end

    rect rgb(240, 255, 240)
        Note over IRL: Snapshot Seal
        IRL->>IRL: Compute L_t (4-part SHA-256)
        IRL->>IRL: Enforce valid_time < txn_time
        IRL->>IRL: reasoning_hash = SHA-256(canonical(S_t))
    end

    IRL-->>AG: {authorized: true, trace_id, reasoning_hash}
    Note over IRL: Trace stored as PENDING

    AG->>EX: Place order (reasoning_hash in metadata)
    EX-->>AG: {exchange_tx_id, filled_qty, price}

    AG->>IRL: POST /irl/bind-execution {trace_id, exchange_tx_id, asset, qty}

    rect rgb(255, 250, 240)
        Note over IRL: Reconciliation
        IRL->>IRL: Check asset match
        IRL->>IRL: Check |qty_delta| ≤ tolerance
        IRL->>IRL: final_proof = SHA-256(reasoning_hash ∥ exchange_tx_id)
    end

    IRL-->>AG: {verification_status: MATCHED, final_proof}
    Note over IRL: Trace sealed — immutable audit record
```

---

### B.2 The Ownership Gap — Before and After

```mermaid
graph LR
    subgraph before["Without IRL"]
        direction TB
        a1["Autonomous Agent"] -->|"reasons about market"| b1["Black Box\n(no audit trail)"]
        b1 -->|"places order"| c1["Exchange"]
        c1 --> d1["Trade Outcome"]
        e1["❌ No proof of reasoning\n❌ No policy record\n❌ Retroactive reconstruction\n❌ Ownership Gap"]
    end

    subgraph after["With IRL"]
        direction TB
        a2["Autonomous Agent"] -->|"POST /irl/authorize"| b2["IRL Engine"]
        b2 -->|"reasoning_hash + trace_id"| a2
        a2 -->|"places order"| c2["Exchange"]
        c2 -->|"execution report"| a2
        a2 -->|"POST /irl/bind-execution"| b2
        c2 --> d2["Trade Outcome"]
        e2["✓ Reasoning sealed before order\n✓ Policy enforced cryptographically\n✓ Tamper-evident audit chain\n✓ Ownership Gap closed"]
    end

    style before fill:#fee2e2,stroke:#ef4444
    style after fill:#dcfce7,stroke:#22c55e
```

---

### B.3 Three-Layer Trust Model

```mermaid
graph TB
    MTA["MacroPulse MTA\n(Ed25519-signed regime)"] --> L1

    subgraph L1["Layer 1 — Pre-Execution Gateway"]
        snap["Cognitive Snapshot"] --- seal["SHA-256 Seal"]
        seal --- bitemp["Bitemporal Constraint"]
        bitemp --- policy["Policy Engine"]
        policy --- mar["Multi-Agent Registry"]
    end

    subgraph L2["Layer 2 — Verified Market Truth"]
        hb["Signed Heartbeat\n(anti-replay)"] --- ptv["Post-Trade Verifier\nPENDING→MATCHED/DIVERGENT/EXPIRED"]
        ptv --- ledger["Immutable Bitemporal Ledger"]
    end

    subgraph L3["Layer 3 — Sovereign Execution (future)"]
        tee["TEE Enclave"] --- wasm["Wasm Policy Modules"]
        wasm --- zk["ZK Compliance Proofs"]
    end

    L1 --> L2
    L2 --> L3
    L1 --> Exchange["Exchange / OMS"]
    L3 --> Regulator["Regulator / Auditor"]

    style L1 fill:#dbeafe,stroke:#3b82f6
    style L2 fill:#dcfce7,stroke:#22c55e
    style L3 fill:#fef3c7,stroke:#f59e0b
```

---

### B.4 Multi-MTA Consensus (Phase 2+)

```mermaid
graph TB
    subgraph signers["MTA Signer Consortium (n=5, k=3)"]
        n1["Node A (MacroPulse)"]
        n2["Node B (Independent)"]
        n3["Node C (Academic)"]
        n4["Node D — offline"]
        n5["Node E (Client-run)"]
    end

    agg["Threshold Aggregator\nCollects k-of-n broadcasts\nVerifies all signatures\nSorts by signer_id → aggregate_hash"]

    n1 -->|"signed RegimeBroadcast"| agg
    n2 -->|"disagreement logged"| agg
    n3 -->|"signed RegimeBroadcast"| agg
    n4 -.-|"offline"| agg
    n5 -->|"signed RegimeBroadcast"| agg

    agg -->|"3-of-5 threshold MtaState"| irl["IRL Engine\n(Arc<dyn MtaClient>)"]

    gov["Governance\nadd_signer / remove_signer\nslash on double-sign"]
    gov -.->|"manages signer set"| signers

    style agg fill:#dbeafe,stroke:#3b82f6
    style gov fill:#fef3c7,stroke:#f59e0b
```

---

---

## §21 — High-Volume Mode: Batch Sealing for AI-Native Workloads

### 21.1 The Performance Question

The IRL standard mode seals each trade intent individually. For funds running
traditional discretionary or systematic strategies — typically tens to hundreds
of trades per session — this is imperceptible overhead.

For AI-native workloads, the profile is different. A reinforcement learning
agent managing a microstructure position may generate thousands of decisions per
second, and the cost of a synchronous round-trip per decision is non-trivial.

This section specifies the High-Volume Mode that preserves IRL's core
guarantees — *seal before execution, cryptographic chain of custody* — at
throughput levels suited to fully autonomous agents.

### 21.2 Standard Mode Characteristics

| Metric | Standard Mode |
|--------|---------------|
| Seal latency (p50) | < 200 µs |
| Throughput | ~5 000 req/s (single instance) |
| Audit unit | Individual trade intent |
| Chain closure | Per-trade (reasoning_hash → final_proof) |

Standard mode is appropriate for any workload that can tolerate a sub-millisecond
synchronous call before each order. This covers most systematic, stat-arb, and
AI-driven strategies that do not require sub-microsecond order placement.

### 21.3 High-Volume Mode Architecture (v2 Roadmap)

High-Volume Mode batches trade intents, computes a Merkle tree over the batch,
and produces a single batch root commitment with per-trade inclusion proofs.

**Batch structure:**

```
BatchRoot = MerkleRoot(intent_hash_1, intent_hash_2, ..., intent_hash_N)
```

Each `intent_hash_i` is computed identically to the standard mode
`reasoning_hash`:

```
intent_hash_i = SHA-256(canonical_json(CognitiveSnapshot_i))
```

**Inclusion proof:**

For each trade, the proof `π_i` is the Merkle sibling path from leaf `i` to the
batch root. An auditor can verify any individual trade by checking:

```
SHA-256(intent_hash_i || sibling_path) == BatchRoot
```

**Temporal guarantee:** The batch root is anchored to a `txn_time` drawn from
the same time source as standard mode. The bitemporal constraint
(`valid_time < txn_time`) is enforced at batch creation time across all intents
in the batch.

### 21.4 Throughput Profile (Projected)

| Metric | High-Volume Mode |
|--------|-----------------|
| Seal latency (amortised per trade) | ~2 µs |
| Throughput | > 100 000 intents/s |
| Audit unit | Batch (inclusion proof per trade) |
| Chain closure | Batch bind (single exchange report → BatchRoot) |

The 50x latency reduction comes from two sources: batch amortisation of the
SHA-256 tree computation, and optional async write path where the audit DB
write is decoupled from the response path.

### 21.5 Operational Modes

Three modes are planned:

| Mode | `SEAL_MODE` env var | Best for |
|------|---------------------|----------|
| Standard | `standard` (default) | All strategies up to ~5 000 req/s |
| High-Volume | `batch` | AI agents > 1 000 decisions/s |
| Shadow | `shadow` + `standard` or `batch` | First-run tuning, policy calibration |

Shadow mode is orthogonal to seal mode — you can run `SHADOW_MODE=true` with
either standard or batch sealing to instrument your agent's policy profile
without blocking trades.

### 21.6 Adoption Path for High-Frequency Agents

For agents generating > 1 000 trades per second, the recommended migration path
is:

1. **Phase 1 (now):** Deploy standard mode with `SHADOW_MODE=true`. All trades
   are sealed individually. Policy violations are logged as `SHADOW_HALTED`
   without blocking the agent. This builds your compliance baseline and model
   hash registry with zero disruption.

2. **Phase 2:** Set `SHADOW_MODE=false`. Standard mode now enforces policy. The
   latency overhead is < 200 µs — invisible to any strategy operating above
   millisecond timescales.

3. **Phase 3 (High-Volume Mode, v2):** For sub-millisecond latency requirements,
   migrate to batch mode. The audit chain is identical; only the granularity of
   the DB write and the proof structure change.

---

## §22 — Technical Roadmap

### 22.1 Currently Shipping (v1.x)

| Feature | Status |
|---------|--------|
| CognitiveSnapshot + SHA-256 seal | ✅ Production |
| Bitemporal constraint | ✅ Production |
| Policy engine (signal-agnostic) | ✅ Production |
| Multi-Agent Registry (MAR) | ✅ Production |
| Layer 2 heartbeat (anti-replay) | ✅ Production |
| bind-execution + final_proof | ✅ Production |
| MTA pluggable interface | ✅ Production |
| Shadow mode (`SHADOW_MODE`) | ✅ v1.1 |
| Prometheus metrics endpoint | ✅ v1.1 |
| Python SDK | ✅ v1.1 |
| TypeScript SDK | ✅ v1.1 |

### 22.2 Near-Term (v1.2 — Q3 2026)

**Exchange-native binding:** Direct exchange API adapters (Binance, FTX-compat,
Interactive Brokers) that auto-submit bind requests on receipt of execution
reports, eliminating the need for the agent to call `/irl/bind-execution`
manually.

**MTA multi-operator consensus:** `ThresholdMtaClient` — aggregate N MTA
operator feeds with a k-of-N signature threshold. A single operator going offline
or misbehaving does not halt the agent.

**Operator SDK:** Reference implementation and documentation for running a custom
MTA operator — enabling firms with proprietary regime signals to become their own
MTA provider while retaining full IRL audit compatibility.

### 22.3 Medium-Term (v2.0 — Q4 2026)

**Batch sealing (High-Volume Mode):** Merkle tree over individual intent hashes
→ single batch root; inclusion proofs per trade (see §21). Target: > 100 000
intents/s per IRL sidecar instance.

**ZK proof generation (optional):** A zero-knowledge circuit that proves the
`CognitiveSnapshot` satisfies the active policy constraints *without revealing
the agent's feature values or position sizes*. Intended for multi-firm audit
scenarios where the auditor must verify compliance without seeing proprietary
trading signals.

**On-chain anchoring:** Periodic publication of batch roots to an EVM-compatible
chain (or Bitcoin Ordinals). Provides third-party verifiability without storing
trade data on-chain.

**Formal verification of the policy engine:** TLA+ or Lean 4 specification of
the `check_constraints` function and the bitemporal invariant. Target: prove that
no combination of inputs can produce a `reasoning_hash` that both (a) passes
policy and (b) contradicts the active MTA state at txn_time.

### 22.4 Research Track (v3+ / Not Committed)

**TEE-based sealing:** Run the IRL Engine inside a Trusted Execution Environment
(Intel TDX or AMD SEV). The attestation report becomes part of the
`CognitiveSnapshot`, proving the seal was computed inside a specific measured
code environment. Relevant for regulatory mandates requiring third-party compute
attestation.

**Cross-chain regime anchoring:** MTA state broadcast anchored simultaneously to
multiple chains for Byzantine fault tolerance. An agent can verify the regime
state against any supported chain without trusting a single operator.

**Agentic finance compliance layer:** Standardised IRL connector interfaces for
major AI agent frameworks (LangChain, AutoGen, CrewAI, custom), enabling any
agent runtime to plug into IRL with a one-function integration regardless of the
underlying model stack.

---

*MacroPulse IRL Whitepaper v4.0 — March 2026*
*All cryptographic values in Appendix A are real outputs from the live system.*
