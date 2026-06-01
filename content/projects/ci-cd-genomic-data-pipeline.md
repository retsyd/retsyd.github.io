---
title: "How I built a testing suite for a Nextflow pipeline with CI/CD"
date: 2025-06-23
---

I work with a methylation sequencing pipeline for data processing pipeline at my current company. It takes raw sequencing reads and produces methylation calls: the quantitative measurements that feed every downstream model and product. If this pipeline breaks, everything downstream breaks. If it produces subtly wrong results, every model trained on that data is compromised.

## The Pipeline

The pipeline is written in Nextflow (DSL2), a workflow manager designed for computational genomics. It orchestrates roughly a dozen bioinformatics tools, each running in its own Docker container. The controller (the Nextflow process itself) runs as an ECS task on AWS, which is launched through an API call.

## The Testing Strategy

The first thing I established was a layered testing strategy. Every pull request to `main` triggers the first four levels of tests in CI/CD on Github actions, running in parallel:

### 1. Double check pre-commit hooks

Pre-commit hooks run across the entire codebase on every PR. (Useful when a developer may forget to setup/install pre-commits locally.) These cover:

- **Python linting and formatting** (Ruff) — catches style issues and common errors
- **Type checking** (mypy) — enforces type annotations on the Python modules in the pipeline
- **Security scanning** (Bandit) — static analysis for common Python security issues like hardcoded credentials or unsafe deserialization
- **Nextflow linting** — validates DSL2 syntax across all pipeline modules and config files
- **General hygiene** — trailing whitespace, YAML/TOML validation, large file detection

This catches the majority of trivial issues before any of the following compute-intensive tests run.

### 2. Unit Tests

Python unit tests run inside the pipeline's own Docker container, pulled from ECR. This ensures tests execute in the same environment as production. Coverage reports are uploaded to Codecov for tracking.

Running tests inside the production container is a deliberate choice. It catches dependency mismatches that would slip through if tests ran in a clean CI environment with separately installed packages.

### 3. Integration Tests

Each pipeline module has its own nf-test suite: a testing framework purpose-built for Nextflow. Integration tests run the actual bioinformatics tools against small test datasets stored in S3 in a development account, verifying that each module produces the expected outputs.

These tests run as a matrix build: one parallel job per module, each pulling its container from ECR and executing against the development AWS account. This parallelisation keeps the total test time manageable despite the number of modules.

### 4. Smoke Tests

A final smoke test validates that the full pipeline can be parsed and initialised with manifest inputs. This catches configuration errors, missing parameters, and broken module imports that wouldn't surface in integration tests.

### 5. E2E Test

Once the pipeline is merged and released to a development environment, an end-to-end test on some raw sequencing reads is launched from the CI/CD Github runner as an ECS task on AWS on a staging account. This takes about half an hour to run from start to finish. The final output generated from the pipeline in S3 is compared against a known reference in the Github action. If the output is the same as the reference, the test passes.
