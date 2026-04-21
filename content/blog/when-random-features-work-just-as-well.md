---
title: "When Random Features Work Just as Well"
date: 2025-01-20
tags: ["machine-learning", "feature-selection", "genomics", "dimensionality"]
summary: "On the counterintuitive finding that randomly selecting features from high-dimensional genomic data often matches the performance of careful feature engineering and why that makes mathematical sense."
---

{{< katex >}}

Early in my work building an epigenetic age prediction model, I needed to reduce the input space from roughly 500,000 CpG sites down to something tractable. A colleague spent considerable time on feature engineering, ranking sites by variance, filtering by biological relevance, running correlation analyses. I tried something simpler: randomly sampling a few thousand features.

The results were essentially the same.

## Why It Happens

In DNA methylation data you're firmly in the *p > n* regime: hundreds of thousands of features, hundreds to thousands of samples. The signal isn't concentrated in a small number of sites. Age-related methylation changes occur across thousands of positions throughout the genome, each contributing weakly.

In this setting, a random subset of 10,000 features drawn from 500,000 will, with high probability, contain enough weakly predictive sites to reconstruct the signal almost as well as the full set. The model doesn't need the *best* features, it needs *enough*, and random sampling delivers that reliably.

Train the same Elastic Net on (1) all ~500,000 features, (2) the top 10,000 by correlation with the target, or (3) a random 10,000. The difference between options 2 and 3 is often negligible. Sometimes the random subset edges ahead, likely by avoiding overfitting to the noisiest univariate correlations.

## The Mathematics

**Spurious correlations are inevitable.** With 500,000 features and a few hundred samples, testing at *p < 0.05* yields ~25,000 false positives. Any method that ranks by univariate association is fishing in a pool where signal and noise are thoroughly mixed. Your carefully engineered feature set may just be selecting the loudest noise.

**The signal is diffuse.** If 5% of features carry meaningful age signal, a random draw of 10,000 will contain roughly 500 genuinely informative ones, more than enough for penalised regression.

**The Hughes phenomenon.** With fixed training samples, predictive power first increases with features then deteriorates. There's an optimal feature set size, and both random and engineered subsets of similar size land in the same performance neighbourhood. The binding constraint isn't *which* features, it's *how many* relative to sample size.

## The Practical Lesson

I watched a colleague invest weeks into a multi-stage pipeline: variance filtering, biological annotation filtering, recursive feature elimination. The final model performed within a fraction of a percent of one trained on a random subset that took minutes to generate.

Feature engineering still matters when signals are genuinely sparse, when you have strong prior knowledge, or when the goal is interpretability. But in high-dimensional omics data where the signal is diffuse, random selection is a competitive baseline. If your engineered features don't substantially outperform a random draw, that tells you the signal is spread too broadly for targeted selection to help.

My rule of thumb: always benchmark against random feature selection. The bottleneck in high-dimensional biological data is rarely which features you pick - it's sample size, label quality, and validation strategy.
