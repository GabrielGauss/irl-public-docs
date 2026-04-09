# IRL Engine — Developer Guide

*v1.3 · April 2026*

---

## Contents

1. [Quick Integration](#1-quick-integration)
2. [SDK Reference](#2-sdk-reference)
3. [Error Matrix](#3-error-matrix)
4. [Heartbeat (Layer 2)](#4-heartbeat-layer-2)
5. [Partial Fills and Divergence](#5-partial-fills-and-divergence)
6. [Shadow Mode](#6-shadow-mode)
7. [Sandbox Setup](#7-sandbox-setup)
8. [Environment Variables](#8-environment-variables)
9. [Position Closing During Kill-Switch (reduce_only)](#9-position-closing-during-kill-switch-reduce_only)
10. [Position Ledger](#10-position-ledger)
11. [Token Rotation](#11-token-rotation)
12. [Hard Questions](#12-hard-questions)

---

## 1. Quick Integration

Install the SDK:

```bash
pip install irl-sdk
```

The integration has two mandatory calls per trade:

### Step 1 — Authorize (before placing the order)

```python
from irl_sdk import IRLClient, AuthorizeRequest, TradeAction, OrderType

async with IRLClient(
    irl_url="https://irl.macropulse.live",
    api_token="your-token",
    mta_url="https://api.macropulse.live",
) as client:
    req = AuthorizeRequest(
        agent_id="00000000-0000-0000-0000-000000000001",
        model_id="hmm-v3.1",
        model_hash_hex="your-model-hash-hex",
        action=TradeAction.LONG,
        asset="BTC-PERP",
        order_type=OrderType.MARKET,
        venue_id="coinbase",
        quantity=2.0,
        notional=120_000.0,
        notional_currency="USD",
    )
    auth = await client.authorize(req)
    # auth.trace_id — include in your exchange order metadata
    # auth.shadow_blocked — True if SHADOW_MODE intercepted a policy block
```

### Step 2 — Bind (after receiving exchange confirmation)

```python
bind = irl.bind(
    trace_id=auth.trace_id,
    exchange_order_id="EX-98765",
    execution_status="Filled",
    execution_price=61_234.50,
    executed_quantity=2.0,
)
# bind.verification_status — MATCHED / DIVERGENT / ORPHAN
# bind.final_proof — SHA-256 chain closure
```

**Always call bind, even on rejected orders.** A rejected order with
`verification_status=MATCHED` is the correct, compliant outcome — it means
the intent was sealed and the rejection was itself captured.

---

## 2. SDK Reference

### Python SDK (`pip install irl-sdk`)

```bash
pip install irl-sdk
```

#### `IRLClient(irl_url, api_token, mta_url)`

| Parameter | Type | Notes |
|-----------|------|-------|
| `irl_url` | str | IRL Engine base URL, e.g. `"https://irl.macropulse.live"` |
| `api_token` | str | Bearer token from `IRL_API_TOKENS` |
| `mta_url` | str | MTA operator base URL for heartbeat fetch. Omit when `LAYER2_ENABLED=false`. |

Use as an async context manager (`async with IRLClient(...) as client:`).

#### `AuthorizeRequest` fields

| Field | Type | Notes |
|-------|------|-------|
| `agent_id` | str | UUID from `POST /irl/agents` |
| `model_id` | str | Human-readable model name |
| `model_hash_hex` | str | 64-char SHA-256 of model config |
| `action` | TradeAction | `TradeAction.LONG` / `SHORT` / `NEUTRAL` |
| `asset` | str | e.g. `"BTC-USD"` |
| `order_type` | OrderType | `OrderType.MARKET` / `LIMIT` / `STOP` / `TWAP` / `VWAP` |
| `venue_id` | str | Exchange identifier |
| `quantity` | float | Order size |
| `notional` | float | Notional value |
| `notional_currency` | str | e.g. `"USD"` |
| `client_order_id` | str | Optional. Your internal order ID. |

#### `client.authorize(req) → AuthorizeResult`

Fetches a signed heartbeat from `mta_url`, then POSTs to `/irl/authorize`.
Returns `AuthorizeResult(trace_id, reasoning_hash, authorized, shadow_blocked)`.

### TypeScript SDK (`npm install irl-sdk`)

```bash
npm install irl-sdk
```

```ts
import { IRLClient, IRLError, IRLHeartbeatError } from "irl-sdk";

const client = new IRLClient({
  irlUrl: "https://irl.macropulse.live",
  apiToken: process.env.IRL_API_TOKEN!,
  mtaUrl: "https://api.macropulse.live",
  timeoutMs: 5_000,
});

// Authorize — heartbeat fetched automatically
const auth = await client.authorize({
  agent_id: "00000000-0000-0000-0000-000000000001",
  model_id: "hmm-v3.1",
  model_hash_hex: "your-model-hash-hex",
  action: "Long",      // "Long" | "Short" | "Neutral"
  asset: "BTC-PERP",
  venue_id: "CBSE",
  quantity: 2.0,
  notional: 120_000,
});

// Bind — closes the cryptographic chain
const bind = await client.bindExecution({
  trace_id: auth.trace_id,
  exchange_tx_id: "EX-12345",
  execution_status: "Filled",
  asset: "BTC-PERP",
  executed_quantity: 2.0,
  execution_price: 61_234.50,
});

await client.close();
```

**Error handling:**
```ts
try {
  const auth = await client.authorize(req);
} catch (err) {
  if (err instanceof IRLHeartbeatError) { /* MTA unreachable */ }
  else if (err instanceof IRLError) { console.error(err.status, err.body); }
}
```

`action` serialisation: `"Long"` → `{ Long: quantity }`, `"Short"` → `{ Short: quantity }`, `"Neutral"` → `"Neutral"`.
Requires Node.js ≥ 18. Exports both CJS and ESM. See [irl-sdk on npm](https://www.npmjs.com/package/irl-sdk).

---

## 3. Error Matrix

All errors return JSON: `{ "error": "ERROR_CODE", "message": "..." }`.

| HTTP | `error` code | Cause | Resolution |
|------|-------------|-------|------------|
| 400 | `BITEMPORAL_VIOLATION` | `valid_time >= txn_time` | Set `valid_time_ms` to the model inference time, not a future timestamp |
| 400 | `HEARTBEAT_MISSING` | No heartbeat provided when `LAYER2_ENABLED=true` | Include a `SignedHeartbeat` in the request |
| 400 | `HEARTBEAT_STALE_SEQUENCE` | `sequence_id ≤` last accepted | Ensure heartbeat sequence is strictly monotone |
| 400 | `HEARTBEAT_DRIFT_EXCEEDED` | Heartbeat older than `MAX_HEARTBEAT_DRIFT_MS` | Send fresh heartbeats; check clock skew |
| 401 | `UNAUTHORIZED` | Invalid or missing `Authorization: Bearer` header | Verify `IRL_API_TOKENS` matches your token |
| 403 | `REGIME_VIOLATION` | Action not in `allowed_sides` for current regime | Agent attempted a direction the MTA prohibits |
| 403 | `NOTIONAL_EXCEEDS_LIMIT` | Cumulative PENDING notional + current request exceeds `agent_cap × mta.max_notional_scale`. The cap is **portfolio-level**: all PENDING (unbound) traces count toward the limit, not just the current request. | Reduce size, or wait for existing PENDING traces to be bound (MATCHED/DIVERGENT) |
| 409 | `TRACE_ALREADY_BOUND` | `trace_id` already has a final verification status (MATCHED/DIVERGENT/EXPIRED) | Do not re-bind a completed trace; each trace can only be bound once |
| 403 | `MODEL_HASH_MISMATCH` | Provided `model_hash_hex` ≠ registered hash | Recompute hash from current model config |
| 403 | `REGIME_UNAUTHORIZED` | Agent's `allowed_regimes` excludes current regime | Add regime to agent profile or wait for regime shift |
| 403 | `AGENT_NOT_ACTIVE` | Agent status is `Suspended` | Re-activate agent via `PATCH /irl/agents/:id/status` |
| 403 | `HEARTBEAT_SIGNATURE_INVALID` | Ed25519 signature verification failed | Check heartbeat signing key |
| 403 | `MTA_SIGNATURE_INVALID` | MTA response signature invalid | Contact your MTA operator |
| 404 | `AGENT_NOT_FOUND` | `agent_id` not in MAR | Register the agent first |
| 404 | `TRACE_NOT_FOUND` | `trace_id` not in DB | Verify you are using the correct IRL instance |
| 429 | `RATE_LIMIT_EXCEEDED` | Too many requests | Back off and retry with jitter |
| 502 | `MTA_FETCH_FAILED` | IRL could not reach MTA operator | Check `MTA_URL`; retry after backoff |
| 500 | `DATABASE_ERROR` | Internal storage failure | Check DB connectivity; contact support |

### Error handling pattern (Python)

```python
from irl_client import IRLClient, IRLError

try:
    auth = irl.authorize(action="Long", quantity=2.0, asset="BTC-PERP", notional=120_000)
except IRLError as e:
    if e.error_code == "REGIME_VIOLATION":
        logger.info("Trade blocked by regime policy — not placing order")
    elif e.error_code in ("MTA_FETCH_FAILED", "DATABASE_ERROR"):
        logger.error("IRL infrastructure error: %s", e.message)
        # Fail-safe: do not place order without a valid trace_id
        raise
    else:
        raise
```

---

## 4. Heartbeat (Layer 2)

Layer 2 provides anti-replay and anti-drift protection. When `LAYER2_ENABLED=true`
(the production default), every authorize request must include a `SignedHeartbeat`.

### Heartbeat fields

| Field | Type | Notes |
|-------|------|-------|
| `sequence_id` | u64 | Strictly monotone counter per agent session |
| `timestamp_ms` | u64 | Unix epoch ms at heartbeat creation |
| `regime_id` | u8 | MTA regime ID at time of heartbeat |
| `mta_ref` | string | MTA version string (from latest `fetch_verified()`) |
| `signature` | bytes | Ed25519 signature over the above fields |

### Heartbeat rules

- `sequence_id` must be strictly greater than the last accepted value.
  IRL rejects equal or lower sequences (`HEARTBEAT_STALE_SEQUENCE`).
- `timestamp_ms` must be within `MAX_HEARTBEAT_DRIFT_MS` of `txn_time`
  (default 200 ms). Stale heartbeats are rejected (`HEARTBEAT_DRIFT_EXCEEDED`).
- The signature key must match the key registered with the agent's profile in MAR.

### Development (LAYER2_ENABLED=false)

For local development and testing, set `LAYER2_ENABLED=false` in `.env`.
IRL will substitute a zero-value heartbeat internally. **Never use this in production.**

---

## 5. Partial Fills and Divergence

A partial fill means the exchange executed less quantity than the authorized intent.
IRL detects this via the `bind_size_tolerance` parameter (default: 0.0001 = 0.01%).

### Bind outcomes for common scenarios

| Scenario | `execution_status` | `verification_status` | Notes |
|----------|-------------------|----------------------|-------|
| Full fill, within tolerance | `Filled` | `MATCHED` | Normal flow |
| Partial fill, within tolerance | `Partial` | `MATCHED` | Delta ≤ 0.01% of intent quantity |
| Partial fill, outside tolerance | `Partial` | `DIVERGENT` | delta > intent_quantity × tolerance |
| Rejected order | `Rejected` | `MATCHED` | Rejection is sealed; chain is closed |
| Asset mismatch | any | `DIVERGENT` | Exchange filled a different instrument |
| No bind within expiry window | (timeout) | `EXPIRED` | Check `/irl/orphans` |

### Bind request fields

| Field | Required | Notes |
|-------|----------|-------|
| `trace_id` | yes | UUID from the authorize response |
| `exchange_tx_id` | yes | Exchange order/fill ID |
| `execution_status` | yes | `"Filled"` / `"Rejected"` / `"Partial"` |
| `asset` | no | Asset actually traded — triggers mismatch check if supplied |
| `executed_quantity` | no | Fill quantity — triggers tolerance check if supplied |
| `execution_price` | no | Fill price — stored for forensic PnL correlation |
| `executed_side` | no | `"Long"` or `"Short"` as reported by the exchange. If supplied, verified against the authorized direction. Omit only if exchange does not return side in the fill report. |
| `execution_time_ms` | no | Unix ms of the actual exchange fill. If omitted, IRL wall clock is used. Provide this for accurate forensic timestamps. |

```python
# Full bind with side verification and exchange timestamp
bind = irl.bind(
    trace_id=auth.trace_id,
    exchange_order_id="EX-98765",
    execution_status="Partial",
    asset="BTC-PERP",
    executed_quantity=1.3,
    execution_price=61_234.50,
    executed_side="Long",              # exchange-reported direction
    execution_time_ms=1711234567890,   # exchange fill timestamp
)
# bind.verification_status — MATCHED if 1.3 ≈ 2.0 within tolerance
#                           — DIVERGENT if delta > 0.0002 (0.01% of 2.0)
#                           — DIVERGENT if executed_side ≠ authorized side
```

---

## 6. Shadow Mode

Shadow mode lets you instrument your agent's policy profile without blocking
any trades. Use it during initial rollout or when testing policy changes.

### How it works

Set `SHADOW_MODE=true` in your environment. When a trade would fail policy:

1. IRL logs a `WARN`-level trace with the policy violation details.
2. The trace is persisted with `policy_result = 'SHADOW_HALTED'`.
3. The authorize response includes `"shadow_blocked": true` and `"authorized": true`.
4. The agent places the order normally.
5. Compliance can review violations at `GET /irl/shadow-violations`.

### Interpreting shadow results

```python
auth = irl.authorize(action="Long", quantity=2.0, asset="BTC-PERP", notional=120_000)

if auth.shadow_blocked:
    # In production mode this trade would have been blocked with 403 REGIME_VIOLATION.
    # In shadow mode you proceed, but should log this for compliance review.
    logger.warning("Shadow block: trace_id=%s", auth.trace_id)
```

### Moving from shadow to enforcement

1. Run shadow mode for at least one full market cycle.
2. Review `GET /irl/shadow-violations` — categorise violations as expected,
   noise, or bugs in your agent's logic.
3. Tune `allowed_regimes` and `max_notional` on the agent profile if needed.
4. Set `SHADOW_MODE=false`. Enforcement is now active.

---

## 7. Sandbox Setup

### Prerequisites

- Docker (for PostgreSQL)
- Rust 1.75+ (for building from source) **or** the pre-built Docker image
- Python 3.9+ or Node 18+ (for the SDKs)

### Start a local instance

```bash
# 1. Start PostgreSQL
docker run -d --name irl-postgres \
  -e POSTGRES_DB=irl \
  -e POSTGRES_USER=irl \
  -e POSTGRES_PASSWORD=irl \
  -p 5432:5432 postgres:16

# 2. Copy and fill the env file
cp .env.example .env
# Edit DATABASE_URL, MTA_MODE=mock, IRL_API_TOKENS=eval-token-change-me

# 3. Run the engine
cargo run --release
# or: docker run --env-file .env -p 4000:4000 macropulse/irl-engine:latest
```

### Register an agent

```bash
curl -s -X POST http://localhost:4000/irl/agents \
  -H "Authorization: Bearer eval-token-change-me" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "sandbox-agent",
    "model_hash_hex": "0000000000000000000000000000000000000000000000000000000000000000",
    "max_notional": 1000000,
    "allowed_regimes": null
  }' | jq .
```

### Test an authorize/bind cycle

```bash
AGENT_ID="<id from above>"

# Authorize
TRACE=$(curl -s -X POST http://localhost:4000/irl/authorize \
  -H "Authorization: Bearer eval-token-change-me" \
  -H "Content-Type: application/json" \
  -d "{
    \"agent_id\": \"$AGENT_ID\",
    \"model_hash_hex\": \"0000000000000000000000000000000000000000000000000000000000000000\",
    \"model_id\": \"test\",
    \"prompt_version\": \"v1\",
    \"feature_schema_id\": \"default\",
    \"hyperparameter_checksum\": \"0000000000000000000000000000000000000000000000000000000000000000\",
    \"action\": \"Long\",
    \"quantity\": 1.0,
    \"asset\": \"BTC-PERP\",
    \"notional\": 60000,
    \"order_type\": \"MARKET\",
    \"agent_valid_time\": $(date +%s000)
  }")

TRACE_ID=$(echo $TRACE | jq -r .trace_id)

# Bind
curl -s -X POST http://localhost:4000/irl/bind-execution \
  -H "Authorization: Bearer eval-token-change-me" \
  -H "Content-Type: application/json" \
  -d "{
    \"trace_id\": \"$TRACE_ID\",
    \"exchange_tx_id\": \"EX-TEST-001\",
    \"execution_status\": \"Filled\",
    \"execution_price\": 60000.00,
    \"executed_quantity\": 1.0
  }" | jq .
```

---

## 8. Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `DATABASE_URL` | required | PostgreSQL connection string |
| `IRL_API_TOKENS` | required | Comma-separated bearer tokens |
| `MTA_MODE` | `MacroPulse` | `mock` for local dev; `none` for pure audit rail; `MacroPulse` for full L2 production |
| `MTA_URL` | required (MacroPulse) | URL of the MTA operator endpoint |
| `MTA_PUBKEY_HEX` | required (MacroPulse) | Ed25519 public key hex of the MTA operator |
| `LAYER2_ENABLED` | `true` | Set `false` to disable heartbeat requirement in dev |
| `SHADOW_MODE` | `false` | Set `true` to log violations without blocking |
| `METRICS_ENABLED` | `true` | Set `false` to suppress `/metrics` endpoint |
| `MAX_HEARTBEAT_DRIFT_MS` | `200` | Maximum heartbeat age before rejection |
| `BIND_SIZE_TOLERANCE` | `0.0001` | Quantity tolerance for MATCHED classification |
| `TRACE_EXPIRY_MS` | `3600000` | Time before PENDING trace becomes EXPIRED (1 hour) |
| `PORT` | `4000` | HTTP listener port |
| `TIME_SOURCE` | `System` | `System` (default) or `NtpSynced`. **Note:** `NtpSynced` is a Phase-2 stub — it currently falls back to system clock. A warning is logged at startup. Do not rely on it for attestation. |
| `RATE_LIMIT_PER_SECOND` | `100` | Maximum requests per bearer token per second. Set to `0` to disable. Applies to all protected routes. |
| `MAX_BODY_BYTES` | `1048576` | Maximum allowed request body size in bytes (default 1 MB). Requests exceeding this return `413 Payload Too Large`. |

---

## 9. Position Closing During Kill-Switch (`reduce_only`)

During a `risk_off` regime, the MTA may restrict `allowed_sides` to `["neutral"]` — no new
longs or shorts. Without a safety valve this would prevent an agent from closing an existing
position, trapping it.

Set `reduce_only: true` on any order that closes or reduces an existing position:

```python
# Closing 2 BTC long during a kill-switch regime
auth = irl.authorize(
    action="Short",            # selling to close the long
    quantity=2.0,
    asset="BTC-PERP",
    notional=120_000.0,
    order_type="MARKET",
    client_order_id="close-001",
    reduce_only=True,          # bypasses allowed_sides check
)
```

**What `reduce_only=True` changes:**

| Check | Normal | reduce_only |
|-------|--------|-------------|
| `allowed_sides` direction check | enforced | bypassed |
| Notional cap | enforced | enforced |

The agent asserts reduce-only intent; IRL seals and audits the claim.
The notional cap is still enforced: you cannot exceed your configured limit even when closing.

---

## 10. Position Ledger

IRL maintains a running `net_quantity` per `(agent_id, asset)` pair in the
`irl.agent_positions` table, updated automatically on every MATCHED bind.

```sql
-- View net positions for all agents
SELECT agent_id, asset, net_quantity, updated_at
FROM irl.agent_positions
ORDER BY updated_at DESC;
```

| Column | Notes |
|--------|-------|
| `net_quantity` | Signed net: positive = net long, negative = net short. Updated by delta on each MATCHED fill. |
| `last_trace_id` | Most recent trace that changed this position. |
| `updated_at` | Timestamp of last update. |

The position ledger is updated **only for MATCHED binds** — DIVERGENT or ORPHAN fills
are not applied (the intent and execution disagreed; reconcile manually).

Position deltas are computed from the authorized action:
- `Long` fill → `+executed_quantity`
- `Short` fill → `-executed_quantity`
- `Neutral` or no `executed_quantity` → no change

**The position ledger is informational** — it does not drive trading decisions or enforce
position limits. Use it for monitoring and compliance reporting.

---

## 11. Token Rotation

Bearer tokens are synced from `IRL_API_TOKENS` to the `irl.api_tokens` database table
at startup. The engine checks this table on every request (60-second in-memory cache
so revocation takes effect within 60 seconds — no restart required).

### Revoke a token

```sql
UPDATE irl.api_tokens
SET status = 'revoked'
WHERE token_hash = encode(digest('the-token-value', 'sha256'), 'hex');
```

The token becomes invalid on the next cache refresh (within 60 seconds).

### Rotate a token (zero-downtime)

1. Add the new token to `IRL_API_TOKENS` in your environment and restart once
   (or insert the new hash directly into the DB).
2. Update the agent's SDK config to use the new token.
3. After confirming the new token is working, revoke the old token:

```sql
UPDATE irl.api_tokens
SET status = 'revoked'
WHERE token_hash = encode(digest('old-token', 'sha256'), 'hex');
```

### Inspect token activity

```sql
SELECT client_name, source, status, last_used_at, created_at
FROM irl.api_tokens
ORDER BY last_used_at DESC NULLS LAST;
```

`last_used_at` is updated at most once per minute per token (debounced to avoid
excessive writes on high-frequency flows).

---

## 12. Hard Questions

### Q: What is the latency overhead?

IRL adds <200 µs (p99) to the authorize path, measured end-to-end on the server:

| Component | Cost |
|-----------|------|
| MTA fetch (cached) | ~0 µs |
| MAR lookup (in-memory cache) | ~0 µs |
| Policy evaluation | O(1) |
| SHA-256 seal | ~1–2 µs |
| JSON canonicalization (RFC 8785) | ~50–100 µs |
| Async DB write (non-blocking) | ~0 µs blocking |

The DB write is async and does not block the response. Your order placement proceeds
before the trace is committed. Total add: **<200 µs p99**.

For strategies >1,000 trades/sec see whitepaper §21 (batch sealing roadmap).

---

### Q: Who controls the kill switch?

The MTA operator (e.g., MacroPulse) broadcasts the regime. The MTA controls which
sides are permitted (`allowed_sides`) and the notional scale (`max_notional_scale`).

IRL enforces the broadcast. It does not set the regime itself.

The firm operating IRL can:
- Suspend individual agents via `PATCH /irl/agents/:id/status` — immediate kill-switch
  for a specific strategy, bypasses MTA.
- Set `max_notional=0` on an agent profile — zero-cap blocks all non-zero notional.
- Stop the IRL process — if IRL is unreachable, agents should **halt trading**
  (defense in depth: never trade without a sealed trace).

---

### Q: What happens if IRL goes down?

Your agent should never place an order without a successful `POST /irl/authorize` response
containing a `trace_id`. If IRL is unreachable:

```python
try:
    auth = irl.authorize(...)
except (IRLError, requests.RequestException) as e:
    logger.critical("IRL unreachable — halting trade: %s", e)
    return  # Do NOT place the order
```

IRL being down is a compliance event, not a bypass. Design your agent to fail closed.

---

### Q: What happens if the MTA goes down?

IRL has a 60-second circuit-breaker fallback. If the MTA is unreachable, IRL continues
using the last known regime for up to 60 seconds, logging a warning. After 60 seconds,
IRL **fails closed** — all authorize requests return `502 MTA_FETCH_FAILED` until the
MTA recovers.

This means:
- Brief MTA hiccup (< 60s): trading continues under last known constraints.
- Extended MTA outage (> 60s): IRL halts new authorizations.

Set `MTA_FALLBACK_TTL_SECS` to tune the window (default: 60).

---

### Q: What happens to orders in flight when the regime changes?

Regime changes take effect on the **next authorize call**. Orders already sealed with
a valid `trace_id` under the previous regime are not retroactively invalidated — the
audit record proves the regime at the moment of decision. Bind those orders normally.

For strategies running on MTA ticks: subscribe to the MTA broadcast and re-fetch the
regime before each inference cycle. The 100ms MTA cache means back-to-back calls in
the same tick will not cause double-fetches.

---

### Q: Is my alpha exposed to anyone?

No. IRL stores only:

- **`latent_fingerprint`**: `SHA-256(model_id || prompt_version || feature_schema_id || hyperparameter_checksum)` — a hash of identifiers, not the model itself or its inputs.
- **`mta_hash`**: SHA-256 of the MTA response body — proves which regime data was used, not your signal logic.
- **`execution` block**: asset, quantity, notional, action — the same fields any exchange order contains.

Raw features, model weights, prompt templates, and alpha logic are never transmitted to
or stored by IRL.

---

### Q: How do I handle model updates in production?

Every deployed version of your model has a `model_hash_hex`. To update without downtime:

1. Register the new hash before deploying: `PATCH /irl/agents/:id` with `model_hash_hex=<new hash>`.
2. Deploy the new model binary/config.
3. Confirm the new `compute_model_hash(config)` matches the registered hash.
4. If there is a gap between steps 1 and 2, requests will return `403 MODEL_HASH_MISMATCH`. Plan a brief maintenance window or use shadow mode during the cutover.

---

### Q: Can I run multiple strategies as separate agents?

Yes. Register one agent per strategy (or per deployed version):

```bash
# Strategy A — trend following
curl -X POST /irl/agents -d '{"name": "trend-v1", "model_hash_hex": "...", "max_notional": 500000}'

# Strategy B — mean reversion
curl -X POST /irl/agents -d '{"name": "mean-rev-v2", "model_hash_hex": "...", "max_notional": 200000}'
```

Each agent has its own:
- Notional cap
- Allowed regimes list
- Suspension status (individual kill-switch)
- Full audit trail in the ledger

Use one IRL token per strategy team or deploy unit. Revoking a token suspends all agents
using it. Suspending an agent suspends only that agent.

---

### Q: What does IRL NOT protect against?

IRL proves **intent was sealed before execution**. It does not:

- Prevent the agent from submitting incorrect signals (garbage in, garbage out — the
  garbage is just sealed and auditable).
- Verify that the signal logic itself is sound or appropriate.
- Prevent the firm from registering a bad model hash, disabling enforcement (`SHADOW_MODE=true`),
  or suspending the engine.
- Replace exchange risk controls (pre-trade checks, margin rules, position limits at the venue).

These are design surfaces the **firm** owns. IRL documents exactly what the agent intended
under exactly which constraints. What the firm does with that information is a governance question.
