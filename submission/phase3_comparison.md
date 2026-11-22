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
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->30.6<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.017<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->8.1<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->4<!--/answer--> |

**Explain the timings.** The three wall times above cover the same 60 documents, but not the
same job: the single-process run and the cloud MapReduce are *the same word count*; Spark's
time is for *TF-IDF*, which is more work (two `reduceByKey`s and a `join`). Rank the three and
explain what each is spending its time on. Say which comparison is like-for-like, and what the
Spark number — a heavier job, on one machine — tells you about where the cloud job's time goes.
(~80 words)

<!--answer:timing_explanation-->
Ranking: local 0.017 s < Spark 8.1 s < cloud MapReduce 30.6 s. The like-for-like pair is local vs cloud, because both run the same word count. The local run is pure computation in one process. The cloud job does the same tiny work but pays for 16 function invocations (cold starts, OIDC auth), about 60 Cloud Storage reads/writes for inputs and shuffle files, workflow scheduling, the map/reduce barrier and chaos retry backoff. Spark does more work (TF-IDF with two reduceByKeys and a join) and still finishes almost 4x faster than the cloud job, on one machine. Its 8 s are mostly JVM/scheduling and Python-worker overhead. So the cloud job's time goes to coordination and data movement over the network, not computation.
<!--/answer-->

**What the counts tell you.** From your stage count: how many shuffles did the Spark job have,
and which operations in `spark_tfidf.py` caused them? From your retry count: roughly what
fraction of the 16 task attempts was retried, and is that what `--chaos 0.3` should produce?
(~50 words)

<!--answer:counts_interpretation-->
4 stages means 3 shuffles: the reduceByKey in term_freq (line 31), the reduceByKey in doc_freq (line 42) and the join (line 54). The last stage is the collect. For the retries: 16 tasks needed 20 attempts, so 4 of 20 attempts failed (20%), or 4 of 16 tasks (25%) needed a retry. With chaos 0.3 I would expect about 30% of attempts to fail (about 7 retries). 4 is a bit low, but with only 20 attempts that is normal random variation.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.** Go through `spark_tfidf.py` and the notebook's Spark cells.
Name each RDD operation and say whether it is a transformation or an action. What happened
when you ran `build_tfidf_rdd` — and what happened when you ran `collect`? (~60 words)

<!--answer:transformations_vs_actions-->
Transformations: parallelize (creates the RDD), flatMap and reduceByKey in term_freq, map and reduceByKey in doc_freq, then map, join and map(to_row) in build_tfidf_rdd. Action: collect() (toDebugString only inspects the graph). When I ran build_tfidf_rdd it returned instantly. Spark only recorded the lineage graph, and no task ran. When I ran collect, Spark split the graph into 4 stages and 12 tasks, executed them (8.1 s) and sent the 5975 rows back to the driver.
<!--/answer-->

**Lineage and recovery.** Using the lineage you printed in 2b and the timeline in 3a: if the
worker holding one output partition of the *first* `reduceByKey` died, what would Spark have
to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery
starts. (~50 words)

<!--answer:lineage_recovery-->
The first reduceByKey's output is ShuffledRDD[3], read after the shuffle that ends stage 0. If the worker holding one of its partitions died, Spark would use the lineage to recompute only that partition. It reads that partition's slice of the stage-0 shuffle files again; if those files were lost too, it re-runs only the matching stage-0 map tasks from ParallelCollectionRDD[0]. Then it reruns the downstream tasks in stages 1 and 2 that needed that partition. The other partitions and their shuffle outputs are reused. Recovery starts at the stage 0 → stage 1 shuffle boundary, not from the beginning of the job.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->
Cloud Workflows: the `mr-wordcount` workflow (`workflow.yaml`) with parallel fan-out and try/retry.
<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->
The `mr-worker` Cloud Run function (`main.py`), one invocation per map or reduce task.
<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->
Cloud Storage: the `<project>-eurecomgpt` bucket, holding `input/`, `shuffle/` and `output/` under each job prefix.
<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->
`partition()` in `mapreduce.py`, i.e. `zlib.crc32(word) % num_reducers`, called inside `run_map`.
<!--/answer-->

**The barrier.** There is no explicit "wait for all mappers" step in `workflow.yaml`. Where,
exactly, is the barrier between the map and reduce phases — which construct provides it?
(~30 words)

<!--answer:barrier_location-->
The barrier is the end of the `map_phase` `parallel` step. A parallel step only completes when all its branches finish, so `reduce_phase` (the next step) cannot start until the last map task has returned.
<!--/answer-->

**Idempotence.** The workflow retries a task that failed. Why is that safe with the worker as
written, and what would go wrong if `run_map` *appended* its pairs to the shuffle file instead
of overwriting it? (~60 words)

<!--answer:idempotence-->
run_map always writes the same object names (`<job>/shuffle/split-NNN/part-r.json`) with upload_from_string, which replaces the whole object. A retried map task reads the same input split and produces the same pairs, so it simply overwrites the file with identical content. Running it twice gives the same result as running it once. If run_map appended instead, a task that ran twice (or crashed halfway and was retried) would leave duplicated pairs in the shuffle file. The reducers would then sum those duplicates and the word counts would be too high, so the result would no longer match the local reference.
<!--/answer-->

**The chaos run.** What did you see in the execution log and in Logs Explorer when you ran
with `--chaos 0.3`? Did the job still succeed, and was the final answer still identical to the
local reference? (~50 words)

<!--answer:chaos_observations-->
In the execution view some map/reduce steps failed with HTTP 503 and a body {"error": "simulated worker crash"}, then were retried after the backoff and succeeded. In Logs Explorer I counted 4 mr-worker requests with status 503 and 16 successful (200) ones for that run. The job still SUCCEEDED in 30.6 s, with matches_local: true and 2221 distinct words, the same as the local reference.
<!--/answer-->
