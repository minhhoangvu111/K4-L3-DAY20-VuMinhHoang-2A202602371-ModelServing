# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **2 physical · 4 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 6.0 | 51% |
| 2 | 11.8 | 100% |
| 4 | 10.5 | 89% |

**Best**: `-t 2` at 11.8 tok/s
**Slowest tested**: `-t 1` at 6.0 tok/s (1.97x spread)
**Against the physical-core default** (`-t 2`, 11.8 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=2 make bench
```

## Your explanation

**Knee at `-t 2` (physical core count)** — matches the expected shape for a
memory-bandwidth-bound decode workload.

| Point | Observation | Mechanism |
|--:|---|---|
| `-t 1` → `-t 2` | 6.0 → **11.8** tok/s (**1.97×**) | Second physical core adds real memory bandwidth / compute; almost linear speedup. |
| `-t 2` → `-t 4` | 11.8 → 10.5 tok/s (**−11%**) | Two hyperthreads per core (SMT) fight for the **same memory channels and L1/L2 cache**. Decode is bandwidth-bound, not FLOPs-bound, so extra logical threads add contention instead of throughput. |

i5-5200U has only 2 physical cores / 4 logical. Past the physical count, oversubscription
does not open new DRAM paths — it just schedules more workers onto the same two cores.
That is why the curve peaks at physical cores then drops: classic bandwidth contention,
not a measurement glitch.

**Before/after for REFLECTION §5:** `-t 1` (6.0 tok/s) → `-t 2` (11.8 tok/s) = **1.97×**.
Default for later runs: `LAB_N_THREADS=2`.
