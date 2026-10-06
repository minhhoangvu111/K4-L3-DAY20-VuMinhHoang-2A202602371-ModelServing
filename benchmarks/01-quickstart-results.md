# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=2` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 12180 | 1743 / 3338 | 134.7 / 219.0 | 10007 / 15505 / 15505 | 7.4 |
| UD-Q2_K_XL | 0.39 | 7403 | 2157 / 3357 | 130.5 / 150.4 | 10178 / 12822 / 12822 | 7.7 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.04x faster** than `Q4_K_M` here, for 0.11 GB less on disk.

## Your observation

On this i5-5200U (2P/4L, ~6 GB RAM, CPU-only), **UD-Q2_K_XL is not worth it**.

**Speed:** 2-bit decode is only **1.04×** faster (7.7 vs 7.4 tok/s). TPOT P50 drops
from 134.7 → 130.5 ms — barely measurable. TTFT P50 is actually **worse** on 2-bit
(2157 vs 1743 ms), so shorter weights did not help prefill on this compute-bound
CPU; dequantize cost likely cancels the bandwidth win.

**Size:** saves only **0.11 GB** (0.50 → 0.39 GB) — small relative to ~6 GB RAM.

**Quality (same question: "Explain TTFT and TPOT in one sentence each"):**
- Q4_K_M: coherent full sentences (wrong domain definitions, but readable).
- UD-Q2_K_XL: collapses into repetitive nonsense ("Time-to-Fault of the T of the T…").

**Verdict:** keep **Q4_K_M** as the serving default. The 4% decode gain does not justify
the quality collapse on this machine.
