---
title: "From CSVs to Iceberg: Scaling a Genomics ETL Pipeline for ML Training on a budget"
date: 2025-12-14
tags: ["data-engineering", "apache-iceberg", "aws-athena", "parquet", "genomics"]
summary: "How replacing a CSV-join pipeline with Apache Iceberg and a long-format data model cut an ETL pipeline from ~15 minutes to a minute"
---

We had a data ingestion pipeline to create wide-format matrix that worked perfectly well for hundreds of samples. But scaling this to thousands of samples would have broken it entirely. As a start-up, services like Snowflake and Databricks were out of our budget. So, I had to make the most of comparatively cheaper, native solutions in AWS. This post covers how I used long-formats and Apache Iceberg to build a solution that could scale.

## The Data

The domain is DNA methylation: measuring chemical modifications at specific positions (CpG sites) across the genome. Each sample produces a file with roughly 4 million rows: one row per genomic coordinate, with a beta value (0–100) representing the methylation level at that position. Each file is about 100 MB as a CSV.

The typical output would produce a perculiar **wide-format matrix**: CpG coordinates as rows, samples as columns, beta values as cells.

## The Legacy Approach

The existing pipeline worked like this:

1. Read individual per-sample CSV files from S3 via temporary Athena external tables
2. Run a CTAS (Create Table As Select) query that pivots the data into wide format: CpG loci as rows, samples as columns
3. Write the result as a static Parquet file

This produced a correct output, but had fundamental scaling problems:

**Schema explosion on new samples.** Every new sample added a new column to the wide-format output. Adding one sample meant re-running the entire CTAS, rewriting the full file. The SQL queries themselves grew so long they hit API limits, requiring intermediate Parquet files to be created before a final join.

**No incremental updates.** The output was a static snapshot. One file, fixed schema, fixed set of samples. To add a single sample, you had to recreate it again from scratch.

**Athena performance degrades with wide format.** Like relational databases, Athena is optimised for tall, narrow tables, not matrices with hundreds or thousands of columns. As sample count grew, the wide-format queries became increasingly slow. At scale, the query engine was fighting the shape of the data.

For defined client project work where we were analysing a few hundred samples at most, and we could do a "run once, analyse, archive", this was adequate. For continuous sample ingestion to train ML pipelines, it was not.

## The Architecture

This replaced the wide-format-first approach with a three-stage pipeline:

### 1. Ingest CSVs into Parquet

Rather than querying raw CSVs at analysis time, the first stage converts each sample's CSV into Parquet as soon as it arrives.

### 2. Store in Long Format with Iceberg

The core table uses **long format** rather than wide format:

| cpg_coordinate | sample_id | beta_value |
|---|---|---|
| chr1:100-102 | sample1 | 73.33 |
| chr1:102-104 | sample1 | 100.0 |
| chr1:104-106 | sample1 | 68.91 |

One row per locus per sample. The schema is fixed and it never changes regardless of how many samples exist. Adding a new sample means inserting new rows, not adding new columns.

This table is managed by Apache Iceberg, which provides the metadata and transaction layer on top of Parquet files in S3.

### 3. Generate Wide Format On Demand

When a wide-format matrix is needed for ML training, a single `GROUP BY` aggregation query pivots from long to wide. Athena's distributed engine touches each row exactly once regardless of sample count: no multi-stage joins, no intermediate files.

## Why Iceberg

Iceberg isn't just a storage format. It's a metadata and transaction layer that sits between the query engine (Athena) and the data files (Parquet in S3). The architecture has three layers:

**Catalog** (AWS Glue in our case): holds a single mutable pointer per table: the path to the current metadata file. This is the only thing that changes during a write.

**Metadata Layer**: consists of three levels of files, all written to S3 alongside the data:
- *Metadata file* (.metadata.json): the table's complete state at a point in time. Contains the schema, partition spec, and a reference to a manifest list. Every write (INSERT, DELETE, OPTIMIZE) creates a new metadata file. The catalog atomically swaps its pointer to the new file - this is what makes writes atomic.
- *Manifest list* (one per snapshot S0, S1, ...): a list of all the manifest files that make up this snapshot. Crucially, snapshots share manifest files. When you insert a new sample, only a new manifest file is added for those new data files; all existing manifest files from the previous snapshot are reused. This is how time-travel works without duplicating data.
- *Manifest files*: each describes a subset of Parquet data files: which partition they belong to, their row counts, and min/max statistics for every column. Athena uses these statistics to skip files that can't satisfy a query's WHERE clause without opening them (partition pruning).

**Data layer** — the actual Parquet files, never modified in place. Deletes write a separate delete file; the original data file remains.

This architecture gives us several things that plain Athena-over-Parquet cannot provide:

### ACID Transactions

Concurrent reads and writes are safe. No risk of reading a half-written table or corrupting data with overlapping queries.

### Row-Level Deletes

A customer right-to-erasure request becomes a SQL statement:

```sql
DELETE FROM methylation WHERE sample_id = 'sample1'
```

No ETL pipeline to find the original Parquet file, no manual manifest management, no risk of deleting the wrong data. Iceberg writes a delete file; the original data is preserved until explicitly vacuumed.

### Hidden Partitioning

Iceberg partitions data physically (grouping rows with the same partition key into the same files) but manages the mapping through metadata. Unlike Hive-style partitioning with Athena, I don't need to include partition columns in their `WHERE` clauses for pruning to work. Athena consults the manifest statistics and skips irrelevant files automatically - the metadata layer handles everything.

This means we can change the partitioning strategy (e.g., switching from batch-based to chromosome-based) without rewriting queries or telling users anything changed.

### Time Travel

Every write creates a new snapshot. Previous snapshots remain accessible, making it possible to audit what the table looked like before a deletion. It's useful for regulatory compliance, though not sufficient on its own for data compliance (you'd still need CloudTrail-level audit logging for who, when, and why).

## Performance Results

The benchmark compared the two approaches on the same data: merging 148 samples across into a single wide-format Parquet file for analysis.

| Approach | Time |
|---|---|
| Legacy | **857 seconds (14 min)** |
| Iceberg approach | **62 seconds** |

A **14x speedup** — and the gap widens with sample count because the Iceberg approach doesn't re-read or re-join existing data when new samples are added.

The performance difference comes from three compounding factors:

1. **Parquet vs CSV at read time.** Columnar format with compression and predicate pushdown vs. line-by-line text parsing.
2. **Single aggregation vs. cascading joins.** The wide-format pivot is a single `GROUP BY` over the long-format table, not a chain of pairwise sample joins that grow quadratically.
3. **Metadata-driven file skipping.** Athena uses Iceberg manifest statistics to skip files that can't match the query, scanning only the relevant partition.

## Trade-offs and Limitations

**Iceberg adds operational complexity.** The metadata layer needs to be understood by the team. Concepts like snapshots, manifest files, and vacuum are new to anyone used to "just put Parquet files in S3."

**VACUUM is required for true GDPR compliance.** Row-level deletes mark data as deleted in the metadata, but the underlying Parquet bytes remain until a `VACUUM` operation physically removes expired snapshots. For a right-to-erasure request, you need both the `DELETE` and a subsequent `VACUUM`.

**Wide-format generation is now a query, not a file.** The legacy approach produced a static file that analysts could download and work with offline. The Iceberg approach generates wide format on demand via a query. For ML training pipelines that expect a file as input, this means adding a materialisation step.

**Long-format tables are larger.** Storing `sample_id` on every row is more verbose than a single column header in wide format. Parquet compression mitigates this significantly, but the raw row count scales as samples × loci (in our case, 4 million × sample count).

## Key Takeaways

- **Data model matters** Switching from wide-format-first to long-format-first was the biggest win. Long format eliminates schema changes, enables row-level operations, and turns a quadratic join problem into a linear scan.
- **Invest in format conversion early.** Converting CSVs to Parquet on arrival is a small upfront cost that pays dividends on every downstream query. Don't let raw text formats persist in analytical paths.
- **Iceberg is worth the complexity for mutable data.** If your data only ever grows and is never deleted or updated, plain Parquet with Hive-style partitioning may be sufficient. The moment you need deletes, updates, or schema evolution, Iceberg earns its keep.
