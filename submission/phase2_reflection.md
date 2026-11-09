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
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->646.34<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->5.52<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->483752.7<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->13212.7<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.5067<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
1: 2579.02,
2: 2340.963,
4: 2870.409,
8: 3358.108
the fatest was 2
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
The naive version wastes most of its time in the Python interpreter overhead, doing type-checking and dispatching operations one element at a time. The vectorized NumPy version drops down to compiled C code and uses SIMD (Single Instruction, Multiple Data) CPU instructions. This allows the CPU hardware to process blocks of numbers in a single clock cycle instead of sequentially.
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
The curve flattened and stopped helping past 2 threads. This happens because the free Google Colab runtime only provides 2 physical vCPUs. Spawning 4 or 8 threads doesn't give us more compute power; it just forces the CPU to waste time constantly context-switching between threads. Additionally, threads start fighting for the same limited CPU memory bandwidth to fetch data, causing contention overhead.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
Transformers rely heavily on massive, independent matrix multiplications. GPUs have thousands of small cores (SIMT architecture) designed exactly for this type of dense, highly parallel math. However, a workload with heavy sequential logic, branching, or state dependencies (like traversing a linked list or parsing nested JSON) would not benefit from a GPU and would actually run slower.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
Parallelism stops helping when the overhead of managing it exceeds the compute gains, or when strict hardware limits are hit. 

In the vectorization test, the limit is the CPU's SIMD lane width. We improved from 646.34 ms to 5.52 ms, but we can't go faster without wider CPU registers. 

In the thread sweep, we hit a hard hardware limit: lacking physical cores. Scaling stopped at 2 threads (2340.963 ms) because the Colab VM only has 2 vCPUs. Going to 8 threads just added context-switching and coordination overhead, slowing things down.

Finally, for the GPU experiment, the massive drop to 483752.7 ms (vs CPU's 13212.7 ms) shows great parallelization. Here, parallelism eventually stops helping due to Amdahl's Law (the sequential parts of the training loop) and the data movement bottleneck: transferring batches from the host CPU memory across the PCIe bus to the GPU device memory.
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
I would test different batch sizes on the GPU. If running a batch size of 2 takes almost the same time as a batch of 16, it proves the GPU is starved and we are bound by kernel-launch and data transfer overhead, not compute capacity.
<!--/answer-->
