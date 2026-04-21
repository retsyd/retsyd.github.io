---
title: "CI/CD for a Genomic Data Pipeline: Testing, Security, and Multi-Environment Deployment"
date: 2025-06-23
tags: ["ci-cd", "github-actions", "nextflow", "aws", "docker", "devops"]
summary: "How I built a CI/CD system for a Nextflow methylation sequencing pipeline: from pre-commit linting through four layers of testing to automated promotion across development, staging, and production, all backed by reusable GitHub Actions and container image promotion via ECR."
---

I work with a methylation sequencing pipeline for data processing pipeline at my current company. It takes raw sequencing reads and produces methylation calls: the quantitative measurements that feed every downstream model and product. If this pipeline breaks, everything downstream breaks. If it produces subtly wrong results, every model trained on that data is compromised.

When I joined, the pipeline existed but the CI/CD around it didn't. Deployments were manual, testing was ad hoc, and there was no separation between development and production environments. This post describes the CI/CD system I built to change that.

## The Pipeline

The pipeline is written in Nextflow (DSL2), a workflow manager designed for computational genomics. It orchestrates roughly a dozen bioinformatics tools, each running in its own Docker container. The controller (the Nextflow process itself) runs as an ECS task on AWS, triggered by a Lambda function. Each bioinformatics step runs in a separate container pulled from ECR.

This architecture means a deployment involves multiple container images: one controller image and ten or more module images, each independently versioned. The CI/CD system has to handle all of them.

## The Testing Strategy

The first thing I established was a layered testing strategy. Every pull request to `main` triggers four levels of tests, running in parallel.

### Compliance Checks

Pre-commit hooks run across the entire codebase on every PR. (Useful when a developer may forget to setup/install pre-commits locally.) These cover:

- **Python linting and formatting** (Ruff) — catches style issues and common errors
- **Type checking** (mypy) — enforces type annotations on the Python modules in the pipeline
- **Security scanning** (Bandit) — static analysis for common Python security issues like hardcoded credentials or unsafe deserialization
- **Nextflow linting** — validates DSL2 syntax across all pipeline modules and config files
- **General hygiene** — trailing whitespace, YAML/TOML validation, large file detection

This catches the majority of trivial issues before any of the following compute-intensive tests run.

### Unit Tests

Python unit tests run inside the pipeline's own Docker container, pulled from ECR. This ensures tests execute in the same environment as production. Coverage reports are uploaded to Codecov for tracking.

Running tests inside the production container is a deliberate choice. It catches dependency mismatches that would slip through if tests ran in a clean CI environment with separately installed packages.

### Integration Tests

Each pipeline module has its own nf-test suite: a testing framework purpose-built for Nextflow. Integration tests run the actual bioinformatics tools against small test datasets stored in S3, verifying that each module produces the expected outputs.

These tests run as a matrix build: one parallel job per module, each pulling its container from ECR and executing against the development AWS account. This parallelisation keeps the total test time manageable despite the number of modules.

### Smoke Tests

A final smoke test validates that the full pipeline can be parsed and initialised with manifest inputs. This catches configuration errors, missing parameters, and broken module imports that wouldn't surface in isolated unit or integration tests.

### Container Security Scanning

In parallel with the functional tests, Trivy scans the controller Docker image for HIGH and CRITICAL vulnerabilities. This runs on every PR, blocking merge if fixable vulnerabilities are found. The scan uses a centralised Trivy wrapper action that pins the scanner version across all repositories, so vulnerability detection is consistent and version upgrades happen in one place.

## The Deployment Model

The pipeline deploys across three environments of three AWS accounts: development, staging, and production, with different triggers and gates at each stage.

### Development: Automatic on Merge

Every merge to `main` triggers an automatic deployment to development. The workflow builds the controller image, tags it with `git-{short_sha}` (an immutable tag tied to the exact commit), pushes it to the development ECR registry, and updates the ECS task definition. The Lambda function that triggers pipeline runs automatically picks up the new task definition revision.

There's a deliberate design choice here: the development deployment only updates the ECS task definition - it doesn't touch infrastructure. Infrastructure changes go through a separate Terraform workflow. This decoupling means application deployments are fast (under 2 minutes) and can't accidentally break networking, IAM, or storage configuration.

### Staging: Automatic on Release Tag

Creating a semantic version tag (e.g., `v2.1.0`) triggers the release workflow. This:

1. **Validates the tag** — confirms it's a valid semver tag pointing to a commit that's reachable from `main` (preventing releases from feature branches)
2. **Validates the release manifest** — checks that `image_versions.json` (which pins every module image version) is present and valid
3. **Ensures the image exists** — looks for the `git-{sha}` image in ECR, building it if somehow missing
4. **Retags the image** — adds the semantic version tag to the existing image without rebuilding (the same image digest, just a new tag)
5. **Promotes to staging** — promote the controller image and all module images from the development ECR registry to the staging ECR registry using Skopeo*
6. **Runs end-to-end tests** — executes the full pipeline against staging infrastructure with real test data
7. **Deploys to staging** — updates the staging ECS task definition

*The image promotion step deserves detail. Each AWS environment has its own ECR registry in a separate AWS account. Promoting an image means copying it between registries without rebuilding — preserving the exact image digest. The promotion action uses Skopeo for digest-safe, multi-architecture copies and includes safety checks: it refuses to overwrite an existing tag that points to a different digest, preventing accidental image replacement.

### Production: Manual with Gates

Production deployment is triggered manually via `workflow_dispatch`, selecting the target environment and running from a semantic version tag. The workflow:

1. Validates the tag
2. Promotes all images (controller + modules) from the source registry to the production registry
3. Updates the production ECS task definition

The manual trigger is intentional. Production deployments should be a conscious decision, not an automatic side effect of tagging. GitHub environment protection rules provide an additional approval gate.

## Reusable Actions

A key architectural decision was extracting common CI/CD logic into a shared `.github` repository of reusable composite actions and reusable workflows. This means every pipeline and service in the organisation uses the same building blocks:

- **setup-aws-ecr**: configures OIDC-based AWS credentials and logs into ECR
- **build-and-push**: builds a Docker image with layer caching, optional Trivy scanning, and image pushes (skips if the tag already exists)
- **promote-ecr-image-skopeo**: copies images between ECR registries across AWS accounts
- **retag-image**: adds semantic version tags to existing images without rebuilding
- **run-trivy**: centralised Trivy wrapper with pinned version
- **parse-semver-tag**: validates semantic version tags
- **ecs-task-update**: updates ECS task definitions without touching infrastructure

This shared library means bug fixes and security patches to CI/CD logic propagate automatically to every repository. It also enforces consistency: every team's deployment uses the same image promotion logic, the same security scanning, the same tag validation.

## What This Solved

Before this system:

- Deployments were manual operations
- There was no way to know if a change broke a pipeline module without running the full pipeline on real data
- Container images were rebuilt in each environment, introducing the possibility of non-reproducible builds
- There was no security scanning
- Rolling back meant trying to remember what was deployed before

After:

- Every PR gets automated testing (compliance, smoke, unit and integration) before it can be deployed to development and each release to staging gets an E2E test
- Every container image is scanned for known vulnerabilities
- Deployments are "one-click" (staging/production) or automatic (development)
- The same image digest flows from development through staging to production
- Rolling back means redeploying a previous semantic version tag
- All CI/CD logic is shared and centrally maintained
