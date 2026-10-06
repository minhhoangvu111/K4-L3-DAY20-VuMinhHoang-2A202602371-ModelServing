# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=2` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 7 | 0.14 | 15000 | 49000 | 49000 | 3.6 | 0.0% |
| 50 | 14 | 0.24 | 31000 | 58000 | 58000 | 8.2 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.69x** (34% of linear) |
| P95 latency | **1.18x** |
| Effective concurrency at 50 users | 8.2 vs `--parallel 4` slots (occupancy/slot ratio 2.04) |

**Saturated.** Throughput delivered only 1.69x for 5x the offered load, and effective concurrency (8.2) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

P95 grew no faster than throughput (1.18x vs 1.69x), so this server still has headroom at 50 users.

> **Small sample.** Only 7 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Trust the throughput-scaling
> row over the concurrency row here, and run longer (`-t 3m`) if you want firmer numbers.

## Your reading

**Saturates at/below 50 users** on this i5-5200U (2P cores, `--parallel 4`).

**Convincing number:** offered load rose **5×** but delivered throughput only
**1.69×** (0.14 → 0.24 RPS = 34% of linear). That plateau is saturation — extra
users stop buying throughput.

**Supporting evidence (Little's Law + metrics):**
- Effective concurrency at u50 = **8.2** vs **4 slots** (ratio 2.04) → more
  requests in flight than slots can decode → the surplus is **queue time**, not
  slower FLOPs.
- Under `make metrics` during load-50: `requests_deferred` peaked at **46** and
  `n_busy_slots_per_decode` at **2.82/4** — slots full, queue building.
- P95 only rose **1.18×** (49s → 58s) while RPS rose 1.69×: latency is already
  large from queueing even at u10 (eff. concurrency 3.6 ≈ filling the 4 slots).

**First knob to raise goodput@SLO:** reduce **max output tokens** (serving
`max_tokens` / shorter responses), not `--parallel`.
Reason: decode is ~7–11 tok/s on this CPU; each request holds a slot for many
seconds, so L ≫ slots. Shorter outputs free slots faster → more requests finish
inside the latency SLO. Raising `--parallel` above 4 on a **2-core** machine
would admit more work into the same memory channels and likely worsen per-request
TPOT without adding DRAM bandwidth. Threads are already tuned (`-t 2` = knee).
