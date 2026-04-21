---
title: "How to Impress Someone with a Bioinformatics Pipeline"
date: 2023-04-02
tags: ["bioinformatics", "pipelines", "software-engineering", "reproducibility"]
summary: "What makes a good bioinformatics pipeline? A short, non-technical take on the things that matter — from user requirements to reproducibility to knowing when good enough is good enough."
---

Being able to build pipelines is now a requirement of many bioinformaticians. A pipeline can be as small as putting two commands in a bash script to as large as a complex suite of steps.

But what makes a good one?

This post is short, non-technical and to the point, and is aimed at experienced bioinformaticians, those starting out and those who are just curious.

## The Pipeline Must Be Useable and User-Friendly

Your pipeline has to work and work well for your user. Ultimately, to the user, it doesn't matter whether it's written in bash or a workflow manager, or whether it's modular or a monolith. There are tools that will help you get to the finish line, but they help more with longer-term strategy.

Before diving in, you must do the research:

- What does the user want to do?
- What output do they want to see?
- What inputs do they want to use?
- If it goes wrong, what messages do they need to see?
- Can the user use the command line or do they need a GUI?

## Identify the Potential Pitfalls Early

Once you've gathered the requirements, you need to know where the pipeline will be run and explore potential resource limitations. Specifically:

- How large are the dependency files (e.g. databases) and do these have an impact on how quickly the pipeline can be shared (i.e. internet speeds and storage)?
- How long does the pipeline run for? Do the compute resources allow for this and will it impact delivery?
- How many files and intermediate files will be produced in a given time? Is there enough storage for this and should there be a maximum limit at a given time?

You should communicate clearly with the user about these challenges and work with them to find a solution if necessary (rather than hacking your way through it yourself). Sometimes the user may have alternative suggestions that would be sufficient for their needs, e.g. using a smaller database.

## Flexibility and Robustness

Unless the methods embedded in the pipeline are standard or published, it is common for the user to request changes in the future. If you are building a pipeline with more than a couple of steps, decide what tools you will use to isolate each step (with obvious inputs and outputs) that will make it easy for somebody else to swap in another step without breaking the whole pipeline. Workflow managers are a good option here.

## Reproducibility

A pipeline can be considered useless if the data it generates is not reproducible. Therefore, you must test whether the outputs are as expected. Building a suite of tests (unit tests, regression tests, system tests and continuous integration tests) that can also be automated as part of version control will save you a lot of heartache down the line, such as when you need to make changes to a step but need to keep part of the final output the same. The extent of your testing will depend on the requirements of the pipeline.

## Can the Pipeline Afford to Fail?

Like any software project, the effort to meet the requirements above depends on how reliable a pipeline has to be. For example, if it's going to be used a few times by one collaborator that you have quite close contact with, then you may be able to afford to patch fixes, iterate and release often. If you're working on a pipeline that's going to be run thousands of times in one go, creating crucial reports to public health agencies, then you've got to make sure your first release is the best it can be.

## My Take on Where We Are

It's fair to say that research institutions are behind in software development practices to benefit pipeline development (and even laboratory information management systems) that have created a reproducibility crisis. To help tackle this, I also hope it will become common practice to review and test scientific software before its release and publication. Until funders and institutions fund training and retention of skills to nurture best software practices, we've got a lot of pipelines to fix.
