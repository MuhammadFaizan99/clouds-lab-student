# Phase 3 — Comparison Writeup

Copy this file to `submission/phase3_comparison.md`, fill in every answer slot, and commit it:

```bash
cp phase-3-mapreduce-spark/comparison_template.md submission/phase3_comparison.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers in section 1 must **match `submission/phase3_report.json`** — copy them across, do not
retype from memory; they are checked against it.

---

## 1. Facts from your runs

Numbers only (e.g. `12`, `20.3`, `0.041`).

| | Value |
|---|---|
| Map tasks in the cloud job (`mapreduce.num_splits`) | <!--answer:num_map_tasks-->12<!--/answer--> |
| Reduce tasks in the cloud job (`mapreduce.num_reducers`) | <!--answer:num_reduce_tasks-->4<!--/answer--> |
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->48.5<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.007<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->3.0<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->8<!--/answer--> |

**Explain the timings.** The three wall times above cover the same 60 documents, but not the
same job: the single-process run and the cloud MapReduce are *the same word count*; Spark's
time is for *TF-IDF*, which is more work (two `reduceByKey`s and a `join`). Rank the three and
explain what each is spending its time on. Say which comparison is like-for-like, and what the
Spark number — a heavier job, on one machine — tells you about where the cloud job's time goes.
(~80 words)

<!--answer:timing_explanation-->
The local word count was the fastest, taking only 0.007 seconds because everything ran in one process without network communication. Spark took 3.0 seconds to calculate TF-IDF using two reduceByKey operations and a join. Cloud MapReduce was the slowest at 48.5 seconds because of Cloud Run invocations, Cloud Storage communication, workflow scheduling, and retries. The local and cloud word counts are directly comparable because they perform the same task. Spark shows that most of the cloud execution time comes from distributed system overhead.
<!--/answer-->

**What the counts tell you.** From your stage count: how many shuffles did the Spark job have,
and which operations in `spark_tfidf.py` caused them? From your retry count: roughly what
fraction of the 16 task attempts was retried, and is that what `--chaos 0.3` should produce?
(~50 words)

<!--answer:counts_interpretation-->
The Spark job had 4 stages and 3 shuffle boundaries, caused by two reduceByKey operations and one join. In the chaos run, I observed 8 failed attempts out of 16 original tasks, which is 50%. This is higher than the expected 30%, but it is possible because failures are random and retries can also fail.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.** Go through `spark_tfidf.py` and the notebook's Spark cells.
Name each RDD operation and say whether it is a transformation or an action. What happened
when you ran `build_tfidf_rdd` — and what happened when you ran `collect`? (~60 words)

<!--answer:transformations_vs_actions-->
In spark_tfidf.py, flatMap, map, reduceByKey, and join are transformations because they create new RDDs without immediately processing the data. In the notebook, parallelize creates the initial RDD, while collect() is an action. When I ran build_tfidf_rdd, Spark only built the lineage (execution plan). The actual computation started when I called collect(), which executed the transformations and returned 5,975 TF-IDF rows.
<!--/answer-->

**Lineage and recovery.** Using the lineage you printed in 2b and the timeline in 3a: if the
worker holding one output partition of the *first* `reduceByKey` died, what would Spark have
to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery
starts. (~50 words)

<!--answer:lineage_recovery-->
If a worker lost an output partition from the first reduceByKey, Spark would rebuild that partition using its lineage. Recovery would start from the first reduceByKey shuffle boundary. Spark could reuse the available map-side shuffle files and recompute the missing reduce partition. Other available partitions would not need to be recomputed.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Google Cloud Workflows (mr-wordcount), defined in workflow.yaml, schedules tasks, coordinates map and reduce phases, and handles retries.<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->The Cloud Run function mr-worker executes the map and reduce tasks.<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->Google Cloud Storage stores the input data, intermediate shuffle files, and final output.<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->The partitioning logic in mapreduce.py assigns each word to a reducer using a deterministic hash-based rule.<!--/answer-->

**The barrier.** There is no explicit "wait for all mappers" step in `workflow.yaml`. Where,
exactly, is the barrier between the map and reduce phases — which construct provides it?
(~30 words)

<!--answer:barrier_location-->
The barrier is provided by the parallel map step in workflow.yaml. Google Cloud Workflows waits for all parallel map branches to finish before moving to the reduce phase.
<!--/answer-->

**Idempotence.** The workflow retries a task that failed. Why is that safe with the worker as
written, and what would go wrong if `run_map` *appended* its pairs to the shuffle file instead
of overwriting it? (~60 words)

<!--answer:idempotence-->
Retries are safe because the worker overwrites the same Cloud Storage file with the same results each time. This means running a task again does not create duplicate data. If run_map appended pairs instead of overwriting the shuffle file, a retry could add the same words multiple times. This would produce incorrect word counts and make the final result different from the local reference.
<!--/answer-->

**The chaos run.** What did you see in the execution log and in Logs Explorer when you ran
with `--chaos 0.3`? Did the job still succeed, and was the final answer still identical to the
local reference? (~50 words)

<!--answer:chaos_observations-->
I ran the workflow with --chaos 0.3 to simulate random failures. In the execution history, I observed 8 failed task attempts, including HTTP 503 errors. The workflow automatically retried these failures. Despite this, the job finished successfully in 48.5 seconds. The final output contained 2,221 distinct words and matched the local reference exactly.
<!--/answer-->