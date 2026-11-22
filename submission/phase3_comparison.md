# Phase 3 — Comparison Writeup

## 1. Facts from your runs

Numbers only (e.g. `12`, `20.3`, `0.041`).

| | Value |
|---|---|
| Map tasks in the cloud job (`mapreduce.num_splits`) | <!--answer:num_map_tasks-->12<!--/answer--> |
| Reduce tasks in the cloud job (`mapreduce.num_reducers`) | <!--answer:num_reduce_tasks-->4<!--/answer--> |
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->32.7<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.009<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->7.3<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->12<!--/answer--> |

**Explain the timings.**

<!--answer:timing_explanation-->
From fastest to slowest: local single-process (0.009 s), Spark TF-IDF (7.3 s), and Cloud MapReduce (32.7 s). The only like-for-like comparison is local vs. Cloud MapReduce since both run the same word count. Local runs in memory with zero overhead. Spark runs a heavier TF-IDF job locally, spending time on JVM/task scheduling and local shuffles. Seeing Spark beat Cloud MapReduce despite doing more work proves the cloud job spends almost all its time on distributed overhead: HTTP calls, Cloud Run cold starts, Workflows orchestration, retries, and GCS shuffle I/O.
<!--/answer-->

**What the counts tell you.**

<!--answer:counts_interpretation-->
With 4 stages, the Spark job had 3 shuffles, caused by the two `reduceByKey` calls (`term_freq` and `doc_freq`) and the `join`. In the chaos run, 12 failed attempts were retried across the 16 tasks (12 out of 28 total attempts, ~43%), which is slightly above the expected 30% from `--chaos 0.3` due to small-sample variance.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.**

<!--answer:transformations_vs_actions-->
In `spark_tfidf.py` and the notebook, `flatMap`, `map`, `reduceByKey`, and `join` are transformations (lazy), while `collect` is the only action (eager). Running `build_tfidf_rdd` did no actual computation; it only built the logical DAG/lineage graph. Running `collect` triggered the execution of all stages and pulled the final results back to the driver.
<!--/answer-->

**Lineage and recovery.**

<!--answer:lineage_recovery-->
If a worker holding one output partition of the first `reduceByKey` died, Spark would only recompute that single lost partition using the RDD lineage, starting from the input/map stage right before that first shuffle boundary. All other surviving partitions and already-written shuffle files would be reused without restarting the whole job.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->