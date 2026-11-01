# Phase 2 — Reflection: where does parallelism stop helping?

Copy this file to `submission/phase2_reflection.md`, fill in every answer slot, and commit it:

```bash
cp phase-2-tiny-gpt/reflection_template.md submission/phase2_reflection.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or your threading plot freely.

Numbers in section 1 must **match `submission/phase2_report.json`** — copy them across, do not
retype from memory; they are checked against it.

---

## 1. Your measurements

| | Value |
|---|---|
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->238.189<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->1.176<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->432873.4.4<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->12654.1.1<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.4727<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
1 thread: 2704.821 ms, 2 threads: 2304.067 ms, 4 threads: 2254.587 ms, 8 threads: 2127.388 ms. Fastest was 8 threads.
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the same result (the unit test checks that). Explain where the naive version's time actually goes, and what the vectorized one does differently — in terms of the machine, not the Python.

<!--answer:vectorization_why-->
The naive version iterates over sequence rows in Python, issuing a separate numpy operation per row. Each call dispatches a small kernel, transfers a slice to the CPU's math units, and returns — so the processor constantly starts and stops, and the data barely fits in a cache line before the next Python call resets context. The vectorized version batches the entire sequence into one matrix multiply (Q @ K.T), one softmax, and one @ V. The CPU's AVX2/AVX512 SIMD units process eight or sixteen floats per clock cycle across the full matrix without any Python overhead between rows. Data stays in L2/L3 cache for the whole operation, and the hardware prefetcher can stream ahead. That is why 1.18 ms replaces 238 ms: the arithmetic is the same, but the machine is executing it instead of Python.
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping — or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
The curve drops sharply from 1 to 2 threads (2704 ms to 2304 ms, roughly 400 ms saved) then nearly flattens: 4 threads gives 2254 ms and 8 threads gives 2127 ms, a combined gain of only 177 ms for doubling the thread count twice. The Colab T4 runtime exposes 2 vCPUs, so threads beyond 2 share the same physical cores and gain no extra compute. They still must synchronize at barriers inside each matrix multiply, and that coordination overhead grows with thread count while the available parallelism stays capped by 2 physical cores.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the device changed. Say what about a transformer training step suits a GPU, and name one workload that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
A transformer training step is dominated by large matrix multiplications — Q@K, scores@V, the two feedforward projections, and their gradients. These are embarrassingly parallel: thousands of independent dot products with no serial dependencies between them. The T4's 2560 CUDA cores execute them simultaneously at 34x the CPU rate (432873 ms vs 12654 ms). A workload that would not see this speedup: a character-level RNN whose hidden state at step t depends on step t-1. That sequential dependency means GPU threads cannot run ahead, and the GPU's high-latency memory bus actively hurts single-element reads.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware you do not have, coordination that costs more than it saves, data that has to move — and say which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
Three distinct limits appeared across the experiments.

SIMD vectorization hit a Python-dispatch ceiling. The naive attention loop issued one numpy call per row; the CPU spent most of its 238.19 ms on Python overhead rather than arithmetic. Replacing the loop with a single batched matmul let AVX2 units process the full matrix in 1.18 ms. The limit was work that could not be divided into hardware-visible chunks until expressed as a single library call.

CPU threading hit a core-count ceiling. Gains were real from 1 to 2 threads (2704 ms to 2304 ms) because the runtime has 2 vCPUs. Beyond 2 threads the curve nearly flattens (2127 ms at 8 threads) because extra threads compete for the same 2 physical cores. Coordination at matmul barriers costs more than the phantom parallelism saves.

GPU training hit neither: the transformer's large matmuls map perfectly onto thousands of CUDA cores with no serial dependencies, giving a clean 34x speedup (432873 ms vs 12654 ms). The limit would be workloads with sequential state, which the GPU cannot parallelize.
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
I would run the thread sweep on a VM with 8 physical cores (e.g. an n2-standard-8 on GCP) to separate the "ran out of cores" limit from the "coordination overhead" limit. If the curve keeps falling past 4 threads there, the 2-vCPU ceiling was the binding constraint; if it still flattens around 4, synchronization overhead is the true bottleneck regardless of core count.
<!--/answer-->
