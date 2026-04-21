---
title: "Scaling ML Training for Epigenetic Age Prediction"
date: 2024-11-15
tags: ["machine-learning", "aws", "sagemaker", "mlops", "scikit-learn"]
summary: "How parallelising hyperparameter tuning on SageMaker turned a single-instance grid search into a 100x faster training workflow."
---

My company published a [machine learning model in Nature Aging](https://www.nature.com/articles/s41514-025-00314-0) that predicts age from DNA methylation data collected from facial skin. This post covers the engineering side: scaling the training pipeline to handle hundreds of thousands of genomic features and building the ML toolkit that supports it.

## The Data

The input data comes from enzymatic methyl sequencing of DNA extracted from non-invasive skin samples (forehead tape strips). Each sample produces quality-filtered methylation measurements at roughly 500,000 CpG sites, positions in the genome where a cytosine is followed by a guanine, with beta values ranging from 0 to 100 representing the proportion of methylated DNA at each position.

The training dataset comprised thousands of samples, each with a known chronological age label. The target is a regression problem: predict a person's age from their methylation profile.

Half a million features and thousands of samples is a moderately large tabular dataset. It fits in memory on a single machine, but the computational cost of hyperparameter tuning, fitting dozens or hundreds of model configurations with cross-validation, scales with both dimensions.

## The Training Problem

The model uses Elastic Net regression — a linear model that combines L1 (Lasso) and L2 (Ridge) regularisation. This is a well-suited choice for high-dimensional epigenomic data:

- **L1 regularisation** drives coefficients to exactly zero, performing implicit feature selection. With 500,000 features, most of which carry no age-related signal, sparsity is essential.
- **L2 regularisation** handles multicollinearity. Neighbouring CpG sites are often highly correlated, and Ridge-style shrinkage stabilises the coefficient estimates.
- **The `l1_ratio` parameter** controls the balance between the two penalties, and the **alpha parameter** controls overall regularisation strength. Both need to be tuned.

The initial approach ran `GridSearchCV` on a single EC2 instance: iterating over a grid of `l1_ratio` values and 100 alpha values, with 5-fold cross-validation using `GroupKFold` to prevent data leakage from repeated measurements of the same individual.

This worked, but was slow and expensive. A single grid search across the full hyperparameter space took hours on a large instance, and any change to the feature set, preprocessing, or cross-validation strategy meant re-running the entire search.

## Parallelising on SageMaker

The fix was to decompose the grid search into independent jobs and run them in parallel on SageMaker.

Each combination of hyperparameters is an independent fitting problem. There are no dependencies between grid points. This makes hyperparameter tuning embarrassingly parallel. Instead of running a single `GridSearchCV` on one large instance, the training scripts were modified to:

1. **Partition the hyperparameter grid** into individual configurations
2. **Submit each configuration as a separate SageMaker training job** on a smaller, cheaper instance
3. **Collect results** across all jobs and select the best-performing configuration

SageMaker handles the instance provisioning, container management, and job scheduling. The training code itself didn't change. It's still scikit-learn's `ElasticNet` and `GridSearchCV` under the hood, but the outer loop that iterates over hyperparameter configurations was lifted from a single-machine `for` loop to a distributed submission layer.

The result was a **100x speedup** in hyperparameter tuning time. What previously took hours on a single large instance now completed in minutes across many smaller ones, at lower total cost because the instances run only for the duration of each individual fit.

## Key Takeaways

- **Parallel problems should be treated as such.** Hyperparameter tuning has no inter-job dependencies. Lifting the outer loop from a single machine to a managed compute service (SageMaker) gave a 100x speedup with minimal code changes.
