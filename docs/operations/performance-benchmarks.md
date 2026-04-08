# IRL Engine — Performance Benchmarks

Baseline measurements on a single-node deployment (4 vCPU, 8 GB RAM).
Configuration: `MTA_MODE=mock`, `LAYER2_ENABLED=false`, `KMS_PROVIDER=none`.

---

## Throughput

| Concurrency | Throughput | p50 latency | p99 latency |
|---|---|---|---|
| 100 agents | ~2,800 req/s | 18 ms | 42 ms |
| 500 agents | ~4,100 req/s | 68 ms | 210 ms |
| 1,000 agents | ~4,400 req/s | 148 ms | 520 ms |

Bottleneck is PostgreSQL write throughput, not the Rust application layer. The engine itself adds < 2 ms of processing overhead per authorize call (cryptographic sealing, policy evaluation, DB write are the bulk).

---

## Production Overhead

Live configuration adds latency on top of baseline:

| Feature | Additional p50 overhead |
|---|---|
| Live MTA verification | +5–15 ms (network to MTA endpoint) |
| KMS envelope encryption | +8–20 ms (AWS KMS: ~10 ms; local: ~1 ms) |
| Layer 2 heartbeat validation | +1–3 ms |
| Total (all enabled) | ~10–35 ms additional p50 |

---

## Tuning

The primary scaling lever is `DB_POOL_MAX_CONNECTIONS`:

| Concurrency target | Recommended pool size |
|---|---|
| < 50 concurrent agents | 10 (default) |
| 50–200 concurrent agents | 25 |
| 200–500 concurrent agents | 50 |
| 500+ | 100 + read replica via `DB_READONLY_URL` |

Partition maintenance (`pg_partman_bgw`) runs hourly and has negligible throughput impact.

---

## Load Test

Benchmark script: `bench/authorize.lua` (wrk2-compatible).

```bash
wrk2 -t4 -c100 -d60s -R 1000 \
  -s bench/authorize.lua \
  https://irl.macropulse.live
```

Adjust `-R` (target rate) and `-c` (connections) for your concurrency target.
