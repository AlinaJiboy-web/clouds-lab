<!-- Read the phase-3-mapreduce-spark/README.md for the questions, copy
them across, do not retype from memory; they are checked against it. -->

---

## 1. Facts from your runs

Numbers only (e.g. `12`, `20.3`, `0.041`).

| | Value |
|---|---|
| Map tasks in the cloud job (`mapreduce.num_splits`) | <!--answer:num_map_tasks-->12<!--/answer--> |
| Reduce tasks in the cloud job (`mapreduce.num_reducers`) | <!--answer:num_reduce_tasks-->4<!--/answer--> |
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->45.2<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.01<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->3.9<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->5<!--/answer--> |

**Explain the timings.** The three wall times above cover the same 60 documents, but not the same job: the single-process run and the cloud MapReduce are *the same word count*; Spark's time is for *TF-IDF*, which is more work (two `reduceByKey`s and a `join`). Rank the three and explain what each is spending its time on. Say which comparison is like-for-like, and what the Spark number — a heavier job, on one machine — tells you about where the cloud job's time goes. (~80 words)

<!--answer:timing_explanation-->
Fastest to slowest: local (0.01 s) < Spark (3.9 s) < cloud MapReduce (45.2 s).
The local run is fastest: it processes 60 documents in a single Python process with no network or serialisation overhead — pure in-memory iteration.
Spark is slower because TF-IDF is a heavier job: two `reduceByKey` shuffles plus a `join` force in-memory data redistribution across partitions.
The cloud MapReduce is slowest despite parallelism: 12 Cloud Run cold-starts, HTTP round-trips through the Workflows orchestrator, and two GCS reads/writes per map task dominate.
The local and cloud runs are like-for-like (same word-count job), so the ~4500x slowdown is pure cloud coordination overhead.
Spark is not comparable — it computes TF-IDF on one machine; its shuffle cost is analogous to the GCS I/O that consumes most of the cloud job's wall time.
<!--/answer-->

**What the counts tell you.** From your stage count: how many shuffles did the Spark job have, and which operations in `spark_tfidf.py` caused them? From your retry count: roughly what fraction of the 16 task attempts was retried, and is that what `--chaos 0.3` should produce? (~50 words)

<!--answer:counts_interpretation-->
The Spark job had 3 shuffles, one per stage boundary after the input stage: the `reduceByKey` in `term_freq`, the `reduceByKey` in `doc_freq`, and the implicit shuffle in the `join` inside `build_tfidf_rdd`.
With 5 retries out of 16 task attempts the retry fraction is 5/16 ≈ 31%, which matches the expected 30% failure rate injected by `--chaos 0.3`.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.** Go through `spark_tfidf.py` and the notebook's Spark cells. Name each RDD operation and say whether it is a transformation or an action. What happened when you ran `build_tfidf_rdd` — and what happened when you ran `collect`? (~60 words)

<!--answer:transformations_vs_actions-->
Operations in `spark_tfidf.py`: `flatMap` (transformation), `reduceByKey` x2 (transformation), `map` (transformation).
In the notebook: `sc.parallelize` creates `docs_rdd` (transformation); calling `build_tfidf_rdd(docs_rdd)` chains all four transformations — Spark builds the DAG but executes nothing, returning instantly.
`collect()` is the only action; it triggers execution of all four stages and materialises the result list on the driver.
Running `build_tfidf_rdd` returned immediately (lazy evaluation); running `collect()` launched the full computation: 4 stages, 12 tasks.
<!--/answer-->

**Lineage and recovery.** Using the lineage you printed in 2b and the timeline in 3a: if the worker holding one output partition of the *first* `reduceByKey` died, what would Spark have to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery starts. (~50 words)

<!--answer:lineage_recovery-->
Spark would recompute only the lost partition of Stage 1 (the first `reduceByKey` in `term_freq`).
It can reuse all surviving Stage 1 output partitions and the in-memory input from `sc.parallelize` (already on the driver).
Recovery starts at the Stage 0 to Stage 1 shuffle boundary: Spark re-runs the `flatMap` then `reduceByKey` pipeline for that one partition from the original input data.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Cloud Workflows (`workflow.yaml`) — it fans out parallel map steps, waits for all branches to complete via the sequential `next` step, then fans out reduce steps, and retries failed steps via its built-in retry policy.<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->Cloud Run functions (`mr-worker`) — each HTTP invocation runs one map or reduce task and writes its output to GCS.<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->Cloud Storage (GCS bucket `project-50465ec8-48f4-4544-a11-eurecomgpt`) — stores the input document splits, per-mapper shuffle files, and the final word-count output (`wordcount.json`).<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->The `shuffle()` function in `mapreduce.py` combined with the key-range assignment in `run_mr.py` — keys are sorted and divided into equal ranges, routing each key deterministically to one of the four reducers.<!--/answer-->

**The barrier.** There is no explicit "wait for all mappers" step in `workflow.yaml`. Where, exactly, is the barrier between the map and reduce phases — which construct provides it? (~30 words)

<!--answer:barrier_location-->
The `parallel` step in `workflow.yaml` is the barrier: Cloud Workflows executes all map branches concurrently and advances to the next sequential step (the reduce fan-out) only once every branch has completed successfully.
<!--/answer-->

**Idempotence.** The workflow retries a task that failed. Why is that safe with the worker as written, and what would go wrong if `run_map` *appended* its pairs to the shuffle file instead of overwriting it? (~60 words)

<!--answer:idempotence-->
Retry is safe because `run_map` overwrites the shuffle file: every invocation processes the same input split and produces exactly the same key-value pairs, replacing any previous file at that GCS path — the result is identical whether the task runs once or ten times.
If `run_map` appended instead, a retry would write a second copy of its pairs into the shuffle file; the reducer would then sum duplicate values and return word counts that are 2x (or more) the correct answer — silently wrong output with no error raised.
<!--/answer-->

**The chaos run.** What did you see in the execution log and in Logs Explorer when you ran with `--chaos 0.3`? Did the job still succeed, and was the final answer still identical to the local reference? (~50 words)

<!--answer:chaos_observations-->
With `--chaos 0.3`, several map tasks returned HTTP 500 errors; the Cloud Workflows retry policy caught each failure and re-invoked the same worker with the same split.
Despite the injected failures, the job reached state SUCCEEDED with 2,221 distinct words, identical: True — confirming that idempotent overwrite combined with automatic retries produces a correct result even under 30% task failure.
<!--/answer-->
