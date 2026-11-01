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
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->236.7<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->0.902<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->447549.8<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->12712.8<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.4808<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->1 thread: 2458.759 ms; 2 threads: 2158.384 ms; 4 threads: 1985.086 ms; 8 threads: 2536.098 ms. Four threads were the fastest.<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->The naive implementation uses Python loops to calculate attention one operation at a time. This creates overhead because each operation is handled separately. The vectorized implementation uses NumPy matrix operations, which are executed by optimized OpenBLAS routines using SIMD instructions. These instructions process multiple values together and make better use of the CPU. In my experiment, vectorization reduced the time from 236.7 ms to 0.902 ms, giving a 262.42x speedup.<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->Increasing the number of threads initially improved performance. One thread took 2458.759 ms, while four threads took 1985.086 ms. However, increasing to eight threads made execution slower at 2536.098 ms. The best result was therefore four threads. My Colab runtime has 2 available CPUs. Using 4 or 8 threads creates more threads than available CPUs, so adding more threads does not necessarily provide more processing power. Threads also compete for CPU time, cache and memory bandwidth, and their work must be coordinated. These costs can cancel out the benefits of parallel execution.<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->Transformer training involves many matrix multiplications that can be performed in parallel. A GPU has many processing units designed to handle these calculations efficiently. In my experiment, 100 training steps took 447549.8 ms on the CPU but only 12712.8 ms on the T4 GPU, giving approximately a 35.2x speedup. However, a small workload with very little computation may not benefit because transferring data to the GPU and launching kernels can take more time than the calculations themselves.<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->My experiments showed that parallelism improves performance only when the extra work can be divided efficiently and the available hardware can support it. In the attention experiment, vectorization reduced execution time from 236.7 ms to 0.902 ms, a 262.42x improvement. This happened because SIMD and optimized matrix operations reduced instruction and Python-loop overhead. However, some work still has to be performed sequentially, which limits further speedup.

In the CPU threading experiment, four threads were fastest at 1985.086 ms, but eight threads increased the time to 2536.098 ms. This shows that limited CPU resources, thread coordination, and memory contention can make additional parallelism harmful.

The GPU experiment showed a different advantage. The T4 completed 100 steps in 12712.8 ms compared with 447549.8 ms on the CPU. The GPU handled the model's matrix operations more efficiently. However, for smaller workloads, transferring data between CPU and GPU and launching kernels could reduce this advantage. Overall, parallelism stops helping when hardware, sequential work, coordination, or data movement becomes the main bottleneck.<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->I would test different batch sizes on both the CPU and GPU. This would help identify when GPU acceleration becomes worthwhile. I expect small batches to show less speedup because of GPU launch and data-transfer overhead. If even very small batches showed a large speedup, I would reconsider that explanation.<!--/answer-->
