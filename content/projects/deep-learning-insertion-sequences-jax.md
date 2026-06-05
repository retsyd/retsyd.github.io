---
title: "Deep learning to predict insertion sequences in JAX"
date: 2026-06-05
---

In the last few years, deep learning approaches, particularly transformers, have been increasingly applied to genomic data to generate better predictions and foundation models.

Although you need massive resources to build comprehensive foundation models, anyone can build and train their own models on smaller datasets using free TPUs on Google Colab, or for a moderate monthly subscription fee for heavier models.

Luckily, some work on insertion sequences during my PhD and at the Sanger Institute led me to wonder whether I could apply deep learning to these datasets. Insertion sequences are small (700 - 2500 bp), mobile genetic elements primarily found in bacteria that can relocate to different positions within a genome. These insertion sequences can insert themselves near antimicrobial resistance genes in bacteria, facilitating their transfer between bacterial populations. This contributes to the rapid spread of antibiotic resistance. 

Using a Google colab subscription, I trained a combination of model architectures (including convolution nerual networks, transformers and simple MLPs) to predict insertion sequences. I used a training and validation dataset containing 1798 known insertion sequences from the [ISfinder database](https://github.com/thanhleviet/ISfinder-sequences/tree/master) and 1797 non-insertion sequence regions that flanked antimicrobial resistance genes with no known mobilisable features. I applied the best-performing model to predict insertion sequences identified previously using a tool I [created and published](https://www.microbiologyresearch.org/content/journal/mgen/10.1099/mgen.0.000917), with some surprising results.

You can view the notebook [in Google colab](https://colab.research.google.com/drive/12ay_EHfzoUCJCfMFf4-TGmiD2DudoTxP?usp=sharing). If you want to run the training on the transformer blocks, you will need to be in a GPU high-RAM environment, such as A100 GPU, High RAM (go to Runtime->Change runtime environment). However, at the time of writing, you will need a subscription to use this environment.

_Note: if you get `ValueError: numpy.dtype size changed, may indicate binary incompatibility. Expected 96 from C header, got 88 from PyObject`, restart the session_

Some extra reading:
- [Deep Learning for Biology, by Charles Ravarani and Natasha Latysheva](https://learning.oreilly.com/library/view/deep-learning-for/9781098168025/)
- [Nucleotide Transformer](https://www.nature.com/articles/s41592-024-02523-z)
- [JAX tutorials](https://docs.jax.dev/en/latest/notebooks/thinking_in_jax.html)