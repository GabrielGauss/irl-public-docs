# IRL Engine Beta Test Guide

**Version:** 1.2.0  
**Audience:** IRL sidecar beta testers  
**Estimated time:** 30–60 minutes

---

## What You're Testing

The full `authorize → bind` audit chain:

1. Your agent calls **`POST /irl/authorize`** with a reasoning snapshot
2. IRL seals it, checks regime permissions, returns `reasoning_hash` + `authorization_id`
3. You "place" the order (simulated — no real exchange required)
4. Your agent calls **`POST /irl/bind-execution`** with the exchange confirmation
5. IRL produces a final immutable proof record: `MATCHED` or `DIVERGENT`

All traces land in a bitemporal audit table. Nothing is deleted.

---

## Prerequisites

- Python 3.10+ **or** Node.js 18+
- Your IRL sidecar API key (provided by MacroPulse)
- MacroPulse API key (same key works for both services)

---

## Step 1 — Install the SDK

**Python:**
```bash
pip install irl-sdk
```

**TypeScript:**
```bash
npm install irl-sdk
```

---

## Step 2 — Register Your Agent

Every agent that calls `authorize` must be registered first. This is a one-time setup.

**Python:**
```python
import asyncio, httpx

IRL_BASE = "https://api.macropulse.live"
TOKEN = "YOUR_API_KEY_HERE"

async def register_agent():
    async with httpx.AsyncClient() as client:
        r = await client.post(
            f"{IRL_BASE}/irl/agents",
            headers={"Authorization": f"Bearer {TOKEN}"},
            json={
                "agent_id": "my-gpt4-trader-v1",
                "model_hash": "gpt-4o-2024-11-20",   # any stable identifier
                "notional_cap_usd": 50000,
                "allowed_regimes": ["expansion", "recovery"],
                "allowed_sides": ["LONG", "NEUTRAL"],
                "description": "Beta test agent"
            }
        )
        print(r.status_code, r.json())

asyncio.run(register_agent())
```

**Expected response:** `201 Created` with an `agent_id` echoed back.

---

## Step 3 — Get the Current Regime

IRL enforces regime-based permissions. Your agent should read the current regime before deciding to trade.

```python
async def get_regime():
    async with httpx.AsyncClient() as client:
        r = await client.get(
            "https://api.macropulse.live/v1/regime/current",
            headers={"X-API-Key": TOKEN}
        )
        data = r.json()
        print(f"Regime: {data['regime']}  |  Confidence: {data['confidence']}")
        print(f"Signature: {data['signature'][:20]}…")  # Ed25519 from MTA
        return data["regime"]

asyncio.run(get_regime())
```

**Expected:** One of `expansion`, `recovery`, `contraction`, `crisis`.

---

## Step 4 — Authorize a Trade (the critical call)

```python
from irl_sdk import IRLClient, AuthorizeRequest, TradeAction, OrderType

async def run_authorize():
    async with IRLClient(
        base_url="https://api.macropulse.live",
        api_token=TOKEN,
    ) as client:
        result = await client.authorize(AuthorizeRequest(
            agent_id="my-gpt4-trader-v1",
            asset="ES",                           # E-mini S&P futures
            action=TradeAction.LONG,
            order_type=OrderType.MARKET,
            quantity=1,
            notional_usd=21500.0,
            reasoning_snapshot={
                "model": "gpt-4o-2024-11-20",
                "prompt_summary": "Regime is expansion. RSI(14)=42. Net liquidity expanding. Go long ES 1 contract.",
                "confidence": 0.81,
                "risk_rationale": "Stop at -0.5%, target +1.2%. Within daily VaR limit."
            }
        ))
        
        print(f"Status:           {result.status}")          # APPROVED or BLOCKED
        print(f"Authorization ID: {result.authorization_id}")
        print(f"Reasoning Hash:   {result.reasoning_hash}")  # attach this to your order
        print(f"MTA Ref:          {result.mta_ref}")
        print(f"Shadow Blocked:   {result.shadow_blocked}")  # True if shadow mode would block
        
        return result

result = asyncio.run(run_authorize())
```

**Expected:** `status: "APPROVED"`, a `reasoning_hash` (SHA-256 hex), and an `authorization_id` (UUID).

**If you get `BLOCKED`:** The regime check failed — your agent's `allowed_regimes` doesn't include the current regime. Update your agent registration or wait for a matching regime.

---

## Step 5 — Bind the Execution

After your order fills (real or simulated), call `bind-execution` with the exchange confirmation details. The `authorization_id` links back to the original authorization.

```python
from irl_sdk import BindExecutionRequest

async def run_bind(authorization_id: str, reasoning_hash: str):
    async with IRLClient(
        base_url="https://api.macropulse.live",
        api_token=TOKEN,
    ) as client:
        result = await client.bind_execution(BindExecutionRequest(
            authorization_id=authorization_id,
            exchange_order_id="SIM-ORDER-001",    # your OMS order ID
            executed_asset="ES",
            executed_action="LONG",
            executed_quantity=1,
            executed_price=5312.50,
            execution_timestamp="2026-04-14T15:30:00Z",
            exchange="CME",
            reasoning_hash=reasoning_hash         # must match what was authorized
        ))
        
        print(f"Final Status:   {result.status}")  # MATCHED or DIVERGENT
        print(f"Final Proof:    {result.final_proof}")
        print(f"MTA Ref:        {result.mta_ref}")

asyncio.run(run_bind(result.authorization_id, result.reasoning_hash))
```

**Expected:** `status: "MATCHED"` — confirms the execution matched the authorization.

**To test `DIVERGENT`:** Change `executed_action` to `"SHORT"` or `executed_asset` to something else. The engine will record the divergence without blocking (you have the authorization ID).

---

## Step 6 — Verify the Audit Trail

```python
async def check_trace(authorization_id: str):
    async with httpx.AsyncClient() as client:
        r = await client.get(
            f"{IRL_BASE}/irl/audit",
            headers={"Authorization": f"Bearer {TOKEN}"},
            params={"agent_id": "my-gpt4-trader-v1", "limit": 10}
        )
        traces = r.json()["traces"]
        for t in traces:
            print(f"{t['authorization_id'][:8]}… | {t['status']} | {t['valid_time']}")
```

You should see your `MATCHED` (or `DIVERGENT`) trace with full timestamps, reasoning hash, and mta_ref.

---

## Deliverables Checklist

After completing the test, you've validated:

- [ ] Agent registered successfully
- [ ] Regime signal fetched with live Ed25519 signature
- [ ] `APPROVED` authorization returned with `reasoning_hash`
- [ ] `MATCHED` bind-execution recorded in audit trail
- [ ] `DIVERGENT` case tested (optional but encouraged)
- [ ] Audit trail query returns complete trace history

---

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `403 Forbidden` | Invalid or expired API key | Check your `TOKEN` value |
| `422 BLOCKED: regime_not_allowed` | Current regime not in agent's `allowed_regimes` | Update agent registration |
| `422 BLOCKED: notional_cap_exceeded` | Trade notional > agent's cap | Reduce `notional_usd` or raise agent cap |
| `404 authorization_not_found` | `authorization_id` expired (15 min TTL) | Re-authorize |
| `409 already_bound` | Same `authorization_id` bound twice | Each auth can only be bound once |

---

## Feedback

Once you've run through the flow, please share:

1. **Friction points** — anything confusing or hard to integrate
2. **Missing fields** — what would you want in `AuthorizeRequest` or `AuthorizeResult`?
3. **Regime integration** — is the MacroPulse signal useful for your actual strategy?
4. **Compliance use case** — what report format would you need to hand to your risk team?

Open a GitHub issue or email **hello@macropulse.live**.

---

## Resources

| Resource | Link |
|----------|------|
| Full docs | https://github.com/GabrielGauss/irl-public-docs |
| Python SDK | `pip install irl-sdk` |
| TypeScript SDK | `npm install irl-sdk` |
| API reference | https://github.com/GabrielGauss/irl-public-docs/blob/main/docs/developer-guide.md |
| MacroPulse regime API | https://macropulse.live |
