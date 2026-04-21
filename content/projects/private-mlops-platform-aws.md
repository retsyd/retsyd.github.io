---
title: "Building a Private MLOps Platform on AWS"
date: 2025-09-01
tags: ["machine-learning", "mlops", "aws", "terraform", "mlflow"]
summary: "How I deployed MLflow as a authenticated experiment tracking server on AWS and integrated it into a reusable ML toolkit."
---

When our data science team outgrew ad hoc model training on shared EC2 instances, I built an internal MLOps platform from scratch. A central piece was deploying MLflow as a self-hosted, authenticated experiment tracking service with Terraform and a Python toolkit wrapping it for daily use.

## Why Self-Hosted MLflow

At the beginning, the data team had no experiment tracking at all. Training results lived in notebooks, Google docs, Confluence, or Slack messages. There was no way to reproduce a run, compare model versions, or trace a deployed model back to the data and parameters that produced it.

MLflow was the natural choice: open-source, framework-agnostic, and well-integrated with the Python ML ecosystem the team already used. But a managed MLflow service wasn't an option: the data includes patient-derived biological samples. Self-hosting was the only option that met both the security requirements and our budget.

## Infrastructure

I packaged the entire deployment as a reusable Terraform module: ECS Fargate running the MLflow server, RDS PostgreSQL for experiment metadata, S3 for model artifacts, and Secrets Manager for credentials.

## The Tracking Library

Rather than having scientists write raw `mlflow.log_param()` and `mlflow.log_metric()` calls, I included a tracking library in the ML Toolkit package I built that wraps MLflow with decorators and helpers designed for our specific workflow.

### Initialisation

A single `mlflow_init` function handles all setup: tracking URI, experiment selection, and autologging configuration:

```python
from mitrabio.ml_tools.tracking.mlflow_tools import mlflow_init

mlflow_init(
    tracking_server_host="<analytics url>",
    experiment_name="clock_v3_training",
    autolog=True,
    big_schema=True,
)
```

The `big_schema` flag is critical for our use case. With 500,000 methylation features per sample, MLflow's default autologging tries to capture model signatures and input examples, which fails or produces enormous payloads. When `big_schema=True`, the library disables signature inference, input example logging, and dataset metadata logging. Instead, it logs a compressed JSON schema artifact containing column names and dtypes. The model itself is logged with a dummy placeholder signature.

### Tracking Decorators

The library provides decorators that wrap training and evaluation functions with MLflow tracking:

```python
from mitrabio.ml_tools.tracking.mlflow_tools import mlflow_track_regressor

@mlflow_track_regressor(run_name="elasticnet_cv", big_schema=True, evaluate=True)
def train_model(X_train, y_train, X_test, y_test):
    model = ElasticNetCV(l1_ratio=0.5, n_alphas=100, cv=5)
    model.fit(X_train, y_train)
    return model
```

The decorator handles:

1. **Run lifecycle**: creates a nested MLflow run, captures the return value
2. **Model detection**: inspects whether the returned object is a fitted estimator or a `GridSearchCV` result (extracting `best_estimator_` if so)
3. **Model logging**: logs the model with a placeholder signature when `big_schema=True`, and saves the input schema as a compressed artifact
4. **Evaluation**: after the run closes, runs `mlflow.models.evaluate` against the test set, producing regression or classification metrics and diagnostic plots

### Model Retrieval

The library includes helpers for locating logged models by run ID or model ID, returning the S3 artifact path needed for downstream scoring pipelines:

```python
from mitrabio.ml_tools.tracking.mlflow_tools import get_model_s3_path

artifact_path, run_id = get_model_s3_path(run_id="abc123")
```

This is used by the production scoring pipeline to load approved model versions without needing to know the underlying S3 structure.
