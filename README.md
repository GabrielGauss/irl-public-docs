# MacroPulse IRL — Public Documentation

**IRL (Immutable Reasoning Log)** is a cryptographic pre-execution compliance gateway for autonomous AI trading agents. Every trade decision is sealed with SHA-256 before it reaches the exchange, producing a tamper-evident audit trail that maps directly to MiFID II, EU AI Act, and SEC Rule 15c3-5 requirements.

- **Website:** [macropulse.live/irl](https://macropulse.live/irl.html)
- **Live API sandbox:** [irl.macropulse.live](https://irl.macropulse.live)
- **Interactive docs (Swagger UI):** [irl.macropulse.live/swagger-ui/](https://irl.macropulse.live/swagger-ui/)
- **Licensing:** gabriel.veron134@gmail.com

---

## What IRL Does

IRL sits between your AI agent and the exchange. Before any order is placed, the agent calls `POST /irl/authorize` with its full reasoning snapshot. IRL:

1. Runs all pre-execution policy checks (position limits, regime constraints, agent registry)
2. Seals the snapshot with SHA-256 into an immutable `reasoning_hash`
3. Returns the hash to the agent, which embeds it in the exchange order metadata

After execution, the agent calls `POST /irl/bind-execution` with the exchange's `tx_id`. IRL computes `final_proof = SHA-256(reasoning_hash ‖ exchange_tx_id)` — the chain is closed and independently verifiable by any auditor.

---

## Documentation

| Document | Description |
|---|---|
| [Whitepaper](docs/whitepaper.md) | Full IRL protocol specification, cryptographic design, bitemporal model |
| [Getting Started](docs/getting-started.md) | L1 setup in under a day, Docker, environment variables |
| [Developer Guide](docs/developer-guide.md) | SDK usage (Python, TypeScript, Go), API reference, error codes |
| [Case Studies](docs/case-studies.md) | Equity fund, prop desk, quant fund deployment examples |
| [Exchange Integration](docs/exchange-integration.md) | FIX protocol tags, REST order metadata, venue-specific notes |
| [Pricing](docs/pricing.md) | L1/L2/L3 editions, per-agent pricing, volume discounts |
| [Compliance Guide](docs/compliance/compliance-guide.md) | Regulatory mapping: MiFID II, EU AI Act, SEC 15c3-5, DORA |
| [Regulatory Mapping](docs/compliance/regulatory-mapping.md) | Table: requirement → IRL mechanism |
| [Operations Guide](docs/operations/incident-response.md) | Incident response runbook |
| [SLA](docs/operations/service-level-agreement.md) | Uptime commitments, support tiers |
| [Performance Benchmarks](docs/operations/performance-benchmarks.md) | Throughput, p50/p99 latency at scale |
| [Diagrams](diagrams/) | Architecture diagrams (Mermaid source + PNG) |

---

## Try It Now — No Setup Required

The sandbox at [irl.macropulse.live](https://irl.macropulse.live) has three pre-seeded demo agents:

| Use case | agent_id |
|---|---|
| Crypto | `00000000-0000-4000-a000-000000000001` |
| Equities | `00000000-0000-4000-a000-000000000002` |
| Futures | `00000000-0000-4000-a000-000000000003` |

Open [Swagger UI](https://irl.macropulse.live/swagger-ui/), pick an agent_id, and run the full authorize → bind-execution flow interactively.

---

## Repository Structure

```
irl-public-docs/
├── README.md
├── docs/
│   ├── whitepaper.md
│   ├── getting-started.md
│   ├── developer-guide.md
│   ├── case-studies.md
│   ├── exchange-integration.md
│   ├── pricing.md
│   ├── compliance/
│   │   ├── compliance-guide.md
│   │   └── regulatory-mapping.md
│   └── operations/
│       ├── incident-response.md
│       ├── service-level-agreement.md
│       └── performance-benchmarks.md
├── diagrams/
│   └── (architecture diagrams)
├── CONTRIBUTING.md
└── LICENSE
```

---

## License

Documentation: [CC BY-SA 4.0](LICENSE)
Code examples in documentation: MIT

© 2026 MacroPulse Research
