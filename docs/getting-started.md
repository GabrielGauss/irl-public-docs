# Getting Started — L1 Sidecar

> **Try it without installing anything:** visit the public sandbox at the demo URL,
> open `/swagger-ui/`, and use one of the pre-seeded demo agents
> (`00000000-0000-4000-a000-000000000001` for crypto) to run the full
> authorize → bind-execution flow interactively.

IRL is designed to be incrementally adoptable. You can go from zero to a compliant
pre-execution gateway in under a day, without changing your existing trading bot.

---

## What L1 Gives You

| Capability | L1 |
|---|---|
| Pre-execution policy enforcement | ✓ |
| Cryptographic reasoning seal (SHA-256) | ✓ |
| Bitemporal audit ledger | ✓ |
| Tamper-evident trace per trade | ✓ |
| Multi-Agent Registry (fleet governance) | ✓ |
| Post-trade verifier (MATCHED/DIVERGENT/EXPIRED) | ✓ |
| Layer 2 heartbeat (anti-replay) | optional |
| TEE / Wasm policy isolation | L3 |
| ZK compliance proofs | L3 |

---

## Prerequisites

- PostgreSQL 14+ (standalone, or shared with any existing DB)
- An MTA — three options:
  - **Mock (fastest, no account needed):** set `MTA_MODE=mock` — full engine evaluation with no external dependency
  - **MacroPulse (turnkey reference operator):** set `MTA_URL` and `MTA_PUBKEY_HEX`
  - **Custom MTA:** implement the `MtaClient` trait and map your model output to `risk_level`, `max_notional_scale`, `allowed_sides` (see `src/mta.rs`)
- Docker, or a Rust toolchain if building from source

---

## Step 1 — Configure Environment

Copy `.env.example` to `.env` and fill in four values:

```bash
cp .env.example .env
```

```dotenv
# MTA operator endpoint (MacroPulse reference implementation — swap for your own)
MTA_URL=https://your-mta-operator.com
MTA_PUBKEY_HEX=<64-char hex Ed25519 public key from your MTA operator>

# Your database
DATABASE_URL=postgres://user:pass@localhost:5432/yourdb

# Generate one token per client / fund / strategy
# e.g.: openssl rand -hex 32
IRL_API_TOKENS=your-secret-token-here
```

Everything else has safe defaults:
- `LAYER2_ENABLED=false` — skip heartbeat validation for now
- `BIND_SIZE_TOLERANCE=0.0001` — 0.01% quantity tolerance
- `TRACE_EXPIRY_MS=3600000` — 1 hour to confirm a trade
- `PORT=4000`

---

## Step 2 — Run

**Docker (recommended):**
```bash
docker compose up -d
```

The engine will apply all database migrations automatically on first boot and log:
```
IRL Engine starting on port 4000
Migrations applied
Post-trade verifier started (expiry: 3600s)
Listening on 0.0.0.0:4000
```

**From source:**
```bash
cargo build --release
./target/release/irl-engine
```

---

## Step 3 — Register Your Agent

Each bot that trades must be registered once:

```bash
curl -X POST http://localhost:4000/irl/agents \
  -H "Authorization: Bearer your-secret-token-here" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my-btc-bot",
    "model_hash_hex": "<sha256_of_your_model_config>",
    "max_notional": 500000.0
  }'
```

Save the returned `agent_id`. It uniquely identifies this agent in all future traces.

**Computing `model_hash_hex`:**
```python
import hashlib, json

model_config = {
    "model_id": "my-model-v1.2",
    "prompt_version": "v3",
    "feature_schema": "schema-2026-q1",
    "hyperparameters": {"lookback_days": 60}
}
digest = hashlib.sha256(
    json.dumps(model_config, sort_keys=True).encode()
).hexdigest()
print(digest)  # 64-char hex — use this as model_hash_hex
```

---

## Step 4 — Wrap Your Bot's Order Logic

Before your bot sends an order, call `/irl/authorize`. After the exchange confirms, call `/irl/bind-execution`. Everything else in your bot stays the same.

**Python example:**
```python
import requests, json, time

IRL_URL = "http://localhost:4000"
IRL_TOKEN = "your-secret-token-here"
AGENT_ID = "your-agent-uuid"
MODEL_HASH = "your-model-hash-hex"

def irl_authorize(action, asset, quantity, notional,
                  order_type="MARKET", venue_id="XNAS"):
    """
    action examples:
      Long position:   {"Long": 2.0}
      Short position:  {"Short": 2.0}
      Flat/neutral:    "Neutral"
      Equities Buy:    {"Custom": "Buy"}
      Futures open:    {"Custom": "Open Long"}
    """
    resp = requests.post(f"{IRL_URL}/irl/authorize",
        headers={"Authorization": f"Bearer {IRL_TOKEN}"},
        json={
            "agent_id": AGENT_ID,
            "model_hash_hex": MODEL_HASH,
            "model_id": "my-model-v1.2",
            "prompt_version": "v3",
            "feature_schema_id": "schema-2026-q1",
            "hyperparameter_checksum": "your-hyperparam-hash",
            "action": action,
            "asset": asset,
            "order_type": order_type,
            "venue_id": venue_id,
            "quantity": quantity,
            "notional": notional,
            "notional_currency": "USD",
            "multiplier": 1.0,
            "limit_price": None,
            "client_order_id": f"ord-{int(time.time())}",
            "agent_valid_time": int(time.time() * 1000),
        }
    )
    resp.raise_for_status()
    return resp.json()  # {"authorized": true, "trace_id": "...", "reasoning_hash": "..."}


def irl_bind(trace_id, exchange_tx_id, asset, executed_qty, price):
    resp = requests.post(f"{IRL_URL}/irl/bind-execution",
        headers={"Authorization": f"Bearer {IRL_TOKEN}"},
        json={
            "trace_id": trace_id,
            "exchange_tx_id": exchange_tx_id,
            "execution_status": "Filled",
            "asset": asset,
            "executed_quantity": executed_qty,
            "execution_price": price,
        }
    )
    resp.raise_for_status()
    return resp.json()  # {"verification_status": "MATCHED", "final_proof": "..."}


# --- Your existing bot logic, wrapped ---

auth = irl_authorize({"Long": 2.0}, "BTC-PERP", 2.0, 120_000.0)
if not auth["authorized"]:
    raise RuntimeError("IRL blocked trade")

exchange_result = your_exchange_client.place_order(...)  # unchanged

bind = irl_bind(
    auth["trace_id"],
    exchange_result["order_id"],
    exchange_result["asset"],
    exchange_result["filled_qty"],
    exchange_result["avg_price"],
)
print(f"Trade sealed: {bind['verification_status']} — proof: {bind['final_proof']}")
```

**Rust example (minimal):**
```rust
let auth: serde_json::Value = client
    .post("http://localhost:4000/irl/authorize")
    .bearer_auth(&token)
    .json(&authorize_payload)
    .send().await?
    .json().await?;

let trace_id = auth["trace_id"].as_str().unwrap();

// ... place order on exchange ...

let bind: serde_json::Value = client
    .post("http://localhost:4000/irl/bind-execution")
    .bearer_auth(&token)
    .json(&bind_payload)
    .send().await?
    .json().await?;
```

---

## Step 5 — Verify It's Working

**Browser (recommended for first run):**

Open `http://localhost:4000/` — the landing page links to the interactive Swagger UI.
From there you can run every endpoint without writing any code.
The spec is also available at `http://localhost:4000/openapi.json` for import into Postman or Insomnia.

**curl:**

```bash
# Check health
curl http://localhost:4000/irl/health
# → {"status":"ok"}

# View all pending (unconfirmed) traces
curl http://localhost:4000/irl/pending \
  -H "Authorization: Bearer your-secret-token-here"

# View divergent / expired traces (your compliance dashboard feed)
curl http://localhost:4000/irl/orphans \
  -H "Authorization: Bearer your-secret-token-here"

# Replay a specific trace for audit
curl http://localhost:4000/irl/trace/<trace_id> \
  -H "Authorization: Bearer your-secret-token-here"
```

---

## Adoption Path

Start here, add more when you need it:

```
Day 1:   L1 sidecar running, first agent registered, bot wrapped
Week 1:  Compliance team gets read access to /irl/orphans dashboard
Month 1: Enable LAYER2_ENABLED=true for heartbeat anti-replay
Month 3: Add per-regime notional limits per agent in MAR
Later:   L3 TEE / ZK proofs when regulatory requirements escalate
```

You don't need to commit to the full stack on day one.
The audit chain is valuable from the first trade.

---

## Production Notes

**TLS**
The engine binds plain HTTP on port 4000. In production, place a TLS-terminating
reverse proxy in front:

```nginx
# nginx example
server {
    listen 443 ssl;
    ssl_certificate     /etc/ssl/certs/irl.crt;
    ssl_certificate_key /etc/ssl/private/irl.key;

    location / {
        proxy_pass http://localhost:4000;
        proxy_set_header X-Forwarded-For $remote_addr;
    }
}
```

Traefik and Caddy work identically — point them at `localhost:4000`.
Mutual TLS (mTLS) between the IRL engine and your exchange/OMS is recommended
but out of scope for the sidecar tier.

**Token rotation**
Rotate `IRL_API_TOKENS` before any production deployment.
The engine logs a warning on startup if the default `eval-token-change-me` token
is detected. Use `openssl rand -hex 32` to generate production tokens.
