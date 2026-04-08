# IRL Engine — Exchange Integration Guide

*v1.1 · March 2026*

---

## Contents

1. [Integration Architecture](#1-integration-architecture)
2. [Exchange Field Matrix](#2-exchange-field-matrix)
3. [client_order_id Strategy](#3-client_order_id-strategy)
4. [Partial Fill Handling](#4-partial-fill-handling)
5. [Timing Constraints](#5-timing-constraints)
6. [Adapter Pseudocode](#6-adapter-pseudocode)

---

## 1. Integration Architecture

IRL sits between the agent and the exchange. The sidecar must be called **before**
the order is submitted and **after** the execution report is received.

```
Agent                 IRL Engine              Exchange
  │                       │                      │
  ├──POST /authorize───►  │                      │
  │  {action, quantity,   │  fetch MTA            │
  │   notional, ...}      ├───────────────────────►(MTA operator)
  │                       │  enforce policy       │
  │  ◄─{trace_id,         │                      │
  │     reasoning_hash}───┤                      │
  │                       │                      │
  ├──place order─────────────────────────────────►│
  │  (include trace_id                            │
  │   in order metadata)                          │
  │                       │                      │
  │  ◄──execution report──────────────────────────┤
  │  {exchange_tx_id,     │                      │
  │   filled_qty, price}  │                      │
  │                       │                      │
  ├──POST /bind-execution►│                      │
  │  {trace_id,           │                      │
  │   exchange_tx_id,     │                      │
  │   executed_quantity,  │                      │
  │   execution_price}    │                      │
  │                       │                      │
  │  ◄─{final_proof,      │                      │
  │     verification_     │                      │
  │     status}───────────┤                      │
```

The `trace_id` flows into the exchange as part of the order's client metadata
(client tag, strategy ID, or notes field depending on venue). This creates the
correlation that allows `final_proof` to close the audit chain.

---

## 2. Exchange Field Matrix

### IRL → Exchange (authorize response → order fields)

| IRL field | Exchange field (generic) | Binance (Futures) | Interactive Brokers |
|-----------|--------------------------|-------------------|---------------------|
| `trace_id` | Client order tag | `newClientOrderId` | `orderRef` |
| `execution.action` | Side | `side` = `BUY`/`SELL` | `action` = `BUY`/`SELL` |
| `execution.quantity` | Quantity | `quantity` | `totalQuantity` |
| `execution.order_type` | Order type | `type` (`MARKET`/`LIMIT`/...) | `orderType` |
| `execution.limit_price` | Limit price | `price` | `lmtPrice` |
| `execution.asset` | Symbol | `symbol` | `localSymbol` / `conid` |
| `execution.venue_id` | Venue | (ignored — Binance is the venue) | `exchange` |

### Exchange → IRL (execution report → bind fields)

| Exchange field (generic) | Binance (Futures) | Interactive Brokers | IRL bind field |
|--------------------------|-------------------|---------------------|----------------|
| Exchange order ID | `orderId` | `orderId` | `exchange_tx_id` |
| Client order ID (round-trip) | `clientOrderId` | `orderRef` | Use to look up `trace_id` |
| Fill status | `status` | `status` | `execution_status` |
| Filled quantity | `executedQty` | `filled` | `executed_quantity` |
| Average fill price | `avgPrice` | `avgFillPrice` | `execution_price` |
| Fill timestamp | `updateTime` (ms) | `lastFillTime` | `execution_time` |

### Execution status mapping

| Exchange status | IRL `execution_status` | Notes |
|----------------|------------------------|-------|
| `FILLED` | `Filled` | Full fill |
| `PARTIALLY_FILLED` | `Partial` | Submit bind with `executed_quantity` |
| `CANCELED` / `EXPIRED` | `Rejected` | Bind with qty = 0 or original qty; MATCHED |
| `REJECTED` | `Rejected` | Same as above |
| `NEW` / `PENDING` | — | Wait for final status before binding |

**Never bind on intermediate statuses** (`NEW`, `PENDING_CANCEL`). Wait until
the exchange reports a terminal status (`FILLED`, `PARTIALLY_FILLED`, `CANCELED`,
`EXPIRED`, `REJECTED`).

---

## 3. `client_order_id` Strategy

The `client_order_id` field is the correlation key between the IRL trace and the
exchange execution report. Its management is critical for correct bind calls.

### Recommended approach

Generate the `client_order_id` before calling authorize, then pass it in both:
1. The IRL authorize request (`client_order_id` field)
2. The exchange order submission (vendor-specific client order tag field)

```python
import uuid

# Generate before authorize
client_order_id = f"irl-{uuid.uuid4()}"

# 1. Authorize
auth = irl.authorize(
    action="Long",
    quantity=2.0,
    asset="BTC-PERP",
    notional=120_000,
    client_order_id=client_order_id,
)

# 2. Place order — include client_order_id as the exchange tag
exchange.place_order(
    symbol="BTCUSDT",
    side="BUY",
    quantity=2.0,
    new_client_order_id=client_order_id,  # Binance example
)
```

When the execution report arrives, use `client_order_id` to look up the
`trace_id` from your local state (the IRL engine does not expose a
client_order_id lookup endpoint in v1).

```python
# Lookup trace_id from local mapping
trace_id = order_store[client_order_id]["trace_id"]

# Bind
bind = irl.bind(
    trace_id=trace_id,
    exchange_order_id=execution_report["orderId"],
    execution_status=map_status(execution_report["status"]),
    executed_quantity=float(execution_report["executedQty"]),
    execution_price=float(execution_report["avgPrice"]),
    execution_time_ms=execution_report["updateTime"],
)
```

### `client_order_id` uniqueness

Exchange rules vary:
- **Binance Futures:** `newClientOrderId` is unique per order; can be reused after
  cancellation. Use a UUID prefix to avoid collisions across sessions.
- **Interactive Brokers:** `orderRef` is not globally unique — use a combination
  of account + timestamp + UUID fragment.

IRL stores `client_order_id` in the trace but does not enforce uniqueness — that
is the operator's responsibility.

---

## 4. Partial Fill Handling

### Single partial fill (one execution report)

```python
# Exchange reports: 1.3 of 2.0 BTC filled
bind = irl.bind(
    trace_id=auth.trace_id,
    exchange_order_id="EX-98765",
    execution_status="Partial",
    executed_quantity=1.3,        # Must be provided for divergence detection
    execution_price=61_234.50,
)
# verification_status = MATCHED if |2.0 - 1.3| / 2.0 ≤ 0.0001 (0.01%)
# verification_status = DIVERGENT if the delta exceeds tolerance
```

Default `BIND_SIZE_TOLERANCE` is 0.01% (0.0001). A 35% partial fill (1.3/2.0)
will always be `DIVERGENT` unless you explicitly set a higher tolerance.

**Design choice:** DIVERGENT is not an error. It means the exchange deviated from
the authorized intent. This is captured in the audit trail as expected behaviour
for venues with non-deterministic fill rates.

### Successive partial fills (multiple reports for one order)

Some venues send multiple execution reports for a single order (fill-by-fill
streaming). For IRL v1, bind on the **final** execution report only.

```python
# Accumulate fills locally until terminal status
fills = []

for report in exchange_stream:
    fills.append(report)
    if report["status"] in ("FILLED", "CANCELED", "EXPIRED", "REJECTED"):
        # Final fill — compute total
        total_qty = sum(f["executedQty"] for f in fills)
        avg_price = (
            sum(f["executedQty"] * f["price"] for f in fills) / total_qty
            if total_qty > 0 else 0
        )
        irl.bind(
            trace_id=trace_id,
            exchange_order_id=report["orderId"],
            execution_status=map_status(report["status"]),
            executed_quantity=total_qty,
            execution_price=avg_price,
        )
        break
```

### Sizing tolerance guidance by venue

| Venue type | Recommended `BIND_SIZE_TOLERANCE` | Notes |
|------------|----------------------------------|-------|
| Spot (limit, crypto) | `0.0001` (default) | Integer or minimum lot fills only |
| Perpetual futures | `0.001` | Funding settlement may affect size |
| Equity (lot-based) | `0.01` | Odd lot rounding on partial fills |
| FX (notional-based) | `0.005` | Currency conversion rounding |

---

## 5. Timing Constraints

### Authorize → Place order

The authorize call must complete before the order is placed. There is no IRL
timeout on this sequence — the `trace_id` is valid indefinitely for the bind step.

However, the `valid_time` field is subject to the bitemporal constraint:

```
valid_time (agent's reasoning moment) < txn_time (IRL Engine's system clock)
```

If the agent's `valid_time_ms` is set to the model inference timestamp, and
inference happened several seconds ago, this is fine — the constraint only
rejects future-dated `valid_time` values.

### Place order → Bind execution

The bind must be called within `TRACE_EXPIRY_MS` (default: 3 600 000 ms = 1 hour)
of the authorize call's `txn_time`. After expiry, the trace transitions to
`EXPIRED` and the bind endpoint returns `404 TRACE_NOT_FOUND`.

**Practical limit:** Call bind within seconds of receiving the execution report.
The 1-hour window is a safety net for exchange outages, not an intended delay.

### Anti-replay (Layer 2)

The heartbeat `timestamp_ms` must be within `MAX_HEARTBEAT_DRIFT_MS` (default:
200 ms) of the IRL Engine's `txn_time`. This means:

- The heartbeat must be fresh — generated immediately before the authorize call.
- Clock skew between the agent host and the IRL Engine host must be < 200 ms.
- Use NTP on both hosts. In production, synchronise both to the same NTP pool.

---

## 6. Adapter Pseudocode

A complete exchange adapter that handles the full IRL-aware order lifecycle.

```python
import uuid
import time
from irl_client import IRLClient, IRLError

class IRLAwareOrderManager:
    def __init__(self, irl: IRLClient, exchange):
        self.irl = irl
        self.exchange = exchange
        # trace_id keyed by client_order_id
        self.pending: dict[str, str] = {}

    def submit(self, action, quantity, asset, notional, **kwargs):
        client_order_id = f"irl-{uuid.uuid4()}"

        # 1. Authorize
        try:
            auth = self.irl.authorize(
                action=action,
                quantity=quantity,
                asset=asset,
                notional=notional,
                client_order_id=client_order_id,
                valid_time_ms=kwargs.get("inference_time_ms", int(time.time() * 1000)),
            )
        except IRLError as e:
            # Log the block — do not place the order
            print(f"IRL blocked: {e.error_code} — {e.message}")
            return None

        if auth.shadow_blocked:
            print(f"[SHADOW] Would have been blocked: trace_id={auth.trace_id}")

        # 2. Place order — pass client_order_id to exchange
        self.pending[client_order_id] = auth.trace_id
        self.exchange.place_order(
            symbol=asset,
            side="BUY" if action == "Long" else "SELL",
            quantity=quantity,
            client_order_id=client_order_id,
        )
        return auth.trace_id

    def on_execution_report(self, report):
        client_order_id = report["clientOrderId"]
        trace_id = self.pending.get(client_order_id)
        if trace_id is None:
            print(f"No IRL trace for {client_order_id} — cannot bind")
            return

        # Only bind on terminal statuses
        terminal = {"FILLED", "PARTIALLY_FILLED", "CANCELED", "EXPIRED", "REJECTED"}
        if report["status"] not in terminal:
            return

        status_map = {
            "FILLED": "Filled",
            "PARTIALLY_FILLED": "Partial",
            "CANCELED": "Rejected",
            "EXPIRED": "Rejected",
            "REJECTED": "Rejected",
        }

        bind = self.irl.bind(
            trace_id=trace_id,
            exchange_order_id=str(report["orderId"]),
            execution_status=status_map[report["status"]],
            executed_quantity=float(report.get("executedQty", 0)),
            execution_price=float(report.get("avgPrice", 0)) or None,
            execution_time_ms=report.get("updateTime"),
        )

        del self.pending[client_order_id]
        print(
            f"Bound: trace_id={trace_id} "
            f"status={bind.verification_status} "
            f"proof={bind.final_proof[:16] if bind.final_proof else 'none'}..."
        )
```
