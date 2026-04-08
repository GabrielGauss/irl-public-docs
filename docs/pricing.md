# IRL Pricing

## Principles

Pricing scales with the fleet, not the trade volume. A firm running 3 agents under strict regime controls pays less than one running 50 agents across all venues. L1 has no per-trade cost — the compliance value is in the seal existing, not in counting seals.

---

## L1 — IRL Sidecar

**$500 / agent / month**

Drop-in compliance. Operational in under a day.

- Unlimited authorize + bind-execution calls
- Multi-Agent Registry — fleet identity, model hash verification, per-agent notional caps
- Post-trade verifier — MATCHED / DIVERGENT / EXPIRED lifecycle with async expiry worker
- Cryptographic audit ledger — SHA-256 / RFC 8785 canonical JSON, reproducible by any auditor
- Bitemporal timestamps — proves the agent could not have seen the future
- REST API + PostgreSQL persistence
- SDKs: Python, TypeScript, Go
- Email support

Minimum: 1 agent.

---

## L2 — IRL Audit Platform

**$1,200 / agent / month**

Enterprise compliance with anti-replay and signed market truth.

Everything in L1, plus:

- Signed heartbeats — monotonic sequence + Ed25519, binds each request to a specific MTA broadcast
- Replay prevention — an agent cannot use a historical regime to authorise a present-day action
- MacroPulse MTA integration (or custom MtaClient for proprietary regime pipelines)
- Compliance dashboard — read-only view for compliance officers and regulators
- Forensic replay API — any historical trade reconstructable from its sealed snapshot
- 99.9% engine uptime SLA

Minimum: 3 agents.

---

## L3 — IRL Sovereign Gateway

**Enterprise — quoted per engagement**

For clients where compliance cannot reveal alpha.

Everything in L2, plus:

- TEE execution — policy enforced inside a hardware-attested enclave (Intel TDX / AMD SEV)
- Wasm policy modules — hot-reloadable sandboxed policies, no engine changes required
- ZK compliance proofs — prove policy adherence to a regulator without revealing model or strategy
- Double-Green signing — exchange API key inaccessible unless both MTA and MAR checks pass
- Dedicated support channel + quarterly compliance review

Typical range: $8,000–$25,000 / month. Pricing depends on number of agents, TEE infrastructure requirements, regulatory jurisdiction, and support scope.

---

## Volume Discounts (L1 / L2)

| Fleet size | Discount |
|---|---|
| 1–5 agents | List price |
| 6–15 agents | −15% |
| 16–30 agents | −25% |
| 31+ agents | −35% (negotiated) |

---

## Custom MTA — No Additional Fee

Firms that implement their own `MtaClient` (proprietary regime signal) pay standard L1/L2/L3 pricing. There is no additional fee for signal-agnostic deployments.

## MacroPulse MTA Add-On

For L1 deployments wanting MacroPulse regime data as their Market Truth Anchor without upgrading to L2:

**$300 / month** — includes MTA endpoint access and Ed25519 public key registration.

---

## Contact

gabriel.veron134@gmail.com · [macropulse.live/irl](https://macropulse.live/irl.html)
