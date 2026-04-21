---
title: "Building a Reusable ML Toolkit for Genomic Models"
date: 2025-03-01
tags: ["machine-learning", "mlops", "python", "scikit-learn","mlflow"]
summary: "A Python package with a YAML-driven pipeline builder and a prediction CLI."
---

As the number of ML models at my company grew beyond a single epigenetic age clock, the team kept solving the same problems: handling grouped cross-validation, wiring up evaluation metrics, and getting models from training into production scoring. Each new model started with copy-pasted code from the last one.

I built an internal Python package, the ML Toolkit, to consolidate these into a library of tested, reusable components. The design goal was a toolbox that data scientists configure rather than code: define a pipeline in YAML, point it at data, and get a trained, tracked, evaluated model.

## YAML-Configured Pipelines

The core of the toolkit is a dynamic pipeline builder that constructs scikit-learn `Pipeline` objects from YAML configuration files. Each step is specified as a fully-qualified Python class path with its initialisation arguments:

```yaml
pipeline:
  name: clock_v3
  model_version: "3.0.1"
  steps:
    - name: drop_sex_chromosomes
      class: mitrabio.ml_tools.preprocessing.DropCpgsFromChrom
      init_args:
        chromosomes_to_drop: ["chrX", "chrY"]
    - name: qc_filter
      class: mitrabio.ml_tools.preprocessing.QcFilterCpGs
      init_args:
        min_coverage: 10
    - name: normalise
      class: mitrabio.ml_tools.preprocessing.PeakAlignedNormalizer
      init_args:
        strategy: split_shift
    - name: feature_selection
      class: mitrabio.ml_tools.preprocessing.SelectFromModelPercentile
      init_args:
        estimator:
          class: sklearn.linear_model.ElasticNet
          init_args:
            l1_ratio: 0.5
        percentile: 95
    - name: model
      class: mitrabio.ml_tools.training.ElasticNetCVWithGroups
      init_args:
        n_alphas: 100
        cv: 5
```

A `DynamicClassLoader` handles importing classes from string paths, resolving nested class definitions (like the estimator inside feature selection), and importing callable functions (like custom scoring functions). The `PipelineBuilder` validates the config, instantiates each step, and assembles them into a scikit-learn `Pipeline`.

This means experimenting with a different normaliser, feature selection threshold, or model type is a YAML edit rather than a code change. The same pipeline builder is used for the age clock, melanoma prediction, and every other endpoint the team trains. The builder doesn't care where the class comes from as long as it follows the transformer or estimator interface.

## Prediction CLI

The toolkit ships a CLI entry point (`mitrabio-predict`) for scoring new samples against a trained model. It loads the model from MLflow by run ID, downloads the input schema artifact, validates the input data against the schema, runs the pipeline, and writes predictions.

