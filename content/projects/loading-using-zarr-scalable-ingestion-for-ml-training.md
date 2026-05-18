---
title: "Loading using Zarr: Scalable ingestion for ML training"
date: 2026-04-17
tags: ["zarr", "machine-learning", "data-engineering", "aws-athena", "pyarrow", "genomics"]
summary: "Using a chunked, sharded Zarr cache so that training jobs read wide methylation matrices in seconds instead of minutes."
---

In an [earlier project](/projects/csv-to-iceberg-methylation-analytics) I described how methylation data is naturally wide: tens of millions of CpG sites across hundreds-to-thousands of samples. Storing it that way in [Iceberg](/projects/csv-to-iceberg-methylation-analytics/) is a bad idea, so the warehouse holds it long. But every ML training job wants it wide again. This post is about the loader that bridges the two: a Zarr-backed cache that turns a slow, expensive Athena pivot into a one-time cost.

## The Problem

Training and tuning an epigenetic clock means re-reading the same `(samples × CpGs)` matrix dozens of times — across cross-validation folds, hyperparameter sweeps, feature-selection experiments. Each read of the long-format Iceberg table costs an Athena `GROUP BY` pivot, paid in dollars and wall time. With ~28M CpGs and a few thousand samples, that pivot is also large enough that a naive materialisation to a single Parquet file rewrites hundreds of MB on every change.

The earlier wide-format CTAS solved this for small cohorts but suffered schema explosion: every new sample added a column, every addition rewrote the file. By the time the pipeline was ingesting samples continuously, the cost of rewriting the matrix had eclipsed the cost of reading it.

What I needed was something with three properties:

1. **Append-only.** Adding a sample writes new bytes; it doesn't rewrite existing ones.
2. **Chunked.** A training job that wants 50 samples shouldn't pay to scan 5,000.
3. **Cached.** The second call for the same samples should skip Athena entirely.

Zarr provides (1) and (2). A small ingestion layer on top of Athena UNLOAD provides (3).

## Why Zarr

Zarr is a chunked, compressed, n-dimensional array format. The array is split into fixed-size chunks, each persisted as an independent object (file on disk, key in S3). Reading a slice of the array fetches only the chunks it touches; appending along an axis writes new chunks and leaves the existing ones alone.

For a `(N_samples × ~28M_CpGs)` matrix this maps cleanly onto S3:

- **Chunks small along the sample axis, large along the CpG axis.** Reading "all CpGs for 50 samples" hits a contiguous strip of sample-axis chunks. Reading "50 CpGs for all samples" hits a single CpG-axis chunk. Both common training-time patterns are cheap.
- **Sharding.** A naive chunk layout for a 28M-wide axis would explode the S3 object count. Zarr v3 sharding packs many chunks into a single object — in our case ~800 MB per shard — keeping per-object overhead bounded.
- **Fixed CpG axis.** Appends only extend the sample axis, so existing shards are never rewritten.

The store layout ends up as:

```
beta.zarr/                  # 2D, (n_samples, n_cpgs), float32, NaN missing
cpg_ids.npy                 # 1D, fixed at creation
sample_ids.json             # ordered, one per row of beta.zarr
ingested_batches.json       # {batch_id: [sample_id, ...]}
manifest.json               # n_cpgs, cpg_ids sha256, chunk shape, created_at
```

## Append, atomically-ish

The append path looks straightforward but has to be defensive about partial writes. A Zarr array and a JSON sidecars gets updated per batch:

```python
# 1. Extend zarr arrays (S3 PUTs for new chunks only)
self._open_array("a", name=_BETA_NAME).append(beta, axis=0)

# 2. Rewrite sidecars (small, atomic per PUT but not as a group)
self._write_json(_BATCHES_NAME, new_batches)
```

## The loader: Athena UNLOAD → Arrow scatter → Zarr append

The Zarr store is only the cache. The work of getting bytes out of Iceberg is in the loader. Its `load(sample_ids)` method does this:

1. **Diff against the cache.** Anything already in `store.sample_ids` is skipped.
2. **Resolve sample → batch.** A small Athena lookup against `sample_batch_map` groups the missing samples by their partition.
3. **`UNLOAD` once per value table.** One `UNLOAD WHERE batch_id IN (...) AND sample_id IN (...)` for beta, one for coverage. Partitioned by `batch_id` so each batch's shards land in their own sub-prefix.
4. **Scatter to wide.** Stream the long-format Parquet shards in PyArrow record batches and write each value into its `(row, column)` position in a pre-allocated `(n_samples_in_batch, n_cpgs)` numpy buffer.
5. **Append.** One `store.append(batch_id, ...)` per batch.
6. **Slice.** Return the requested samples from the store, in the requested order.

Subsequent calls for the same samples skip steps 2–5 entirely.

## What this changes

For a training job, the contract becomes: "give me these sample IDs as a wide matrix." First call pays the Athena cost. Every subsequent call — across folds, sweeps, runs, days — reads directly from Zarr in seconds. Adding samples is a single append; the existing shards never move. The result is a cache that an ML pipeline can treat as a local NumPy array even though it's living in S3.
