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
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->307.6<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->12<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->458405<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->12626<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.4502<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
{
    "1": 2218.578,
    "2": 2317.217,
    "4": 2265.765,
    "8": 2601.715
  } 1 thread was fastest
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
Vectorized attention ran 25.6× faster (307.60 ms vs 12.00 ms). In the naive version, each scalar multiply-add carries dozens of extra instructions (interpretation, type checks, object allocation), and operands are scattered in memory, causing cache misses while the SIMD units sit idle. The vectorized version makes one call into AVX-enabled BLAS: contiguous data streams through the cache, each instruction processes several floats at once, and blocking reuses data in registers. Same arithmetic, far more useful work per instruction.
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
It never really got better: 1 thread was fastest (2218.6 ms). 2 threads (2317.2) and 4 threads (2265.8) were slightly slower, and 8 threads (2601.7) was clearly worse. My Colab runtime only has 2 vCPUs, which are probably two hyperthreads on one physical core, so extra threads just compete for the same execution units. Each threaded op also has to split work, sync at a barrier and share memory bandwidth. With matrices this small, that coordination costs more than it saves, and 8 threads adds context switching on top.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
GPU training was about 36× faster (458405 vs 12626 ms). A transformer step is mostly big matmuls and elementwise ops: the same simple math applied to huge arrays with no branching. That maps directly onto thousands of GPU cores and high memory bandwidth. Something sequential and branchy would not speed up, like generating text one token at a time with batch size 1, or a tiny model where kernel launch and host-to-device copies dominate.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
Each experiment hit a different wall. Vectorizing attention gave 25.6× (307.6 → 12.0 ms). The naive version was limited by a serial interpreter instruction stream and scattered data that missed cache. So that was really a data-movement and overhead limit, and once the data was contiguous, SIMD could finally do the work.

Threading hit two limits together: hardware I don't have and coordination cost. With only 2 vCPUs, 8 threads (2601.7 ms) was 17% slower than 1 thread (2218.6 ms). The extra threads had no cores to run on and still paid for syncing.

The GPU gave a big speedup (458405 → 12626 ms), but it is still limited by work that can't be divided. Step n+1 needs the weights from step n, so the 100 steps must run in order. On top of that, every step pays for kernel launches and data movement between host and device.

So parallelism stops helping once the remaining cost is serial dependency, missing hardware, sync overhead, or moving data, rather than arithmetic.
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
Rerun the thread sweep with much bigger matrices (e.g. 4× the hidden size) on a runtime with more physical cores. If 2–4 threads then beat 1, my plateau was mainly coordination overhead on small work, not just missing hardware. If 1 thread still wins, the core count was the real limit.
<!--/answer-->
