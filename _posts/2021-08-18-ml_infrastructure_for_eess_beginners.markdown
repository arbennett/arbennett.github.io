---
layout: post
title:  "Connecting Google Colab compute instances to your data via `fsspec` & `xarray`"
date:   2021-08-18
categories: Software, Machine-learning
---

### Hey y'all: This is going to be a living document for the time being - I'm publishing this as I write, and so this won't be considered "finished" until I take this warning away

It's pretty indisputable that machine learning is both trendy and effective in a number of Earth & environmental
sciences. With the speed and uptake of, particularly, deep-learning methods it's hard to keep up with best practices
for training and deploying models. It's also expensive to roll your own hardware to build a DL platform for these applications,
where data sizes can be large, heterogeneous, and generally difficult to wrangle. That is to say, the data space of Earth science
is moving as fast as machine-learning. In well funded projects this isn't necessarily a problem, but for exploring hypotheses and
proofs-of-concept it's important to have access to hardware that can be useful on limited timelines. Particularly, this can be a
daunting landscape for students who are expected to learn the science and technology concurrently.

Okay, passive voice aside, this is a tough space to operate in. GPUs are absurdly expensive at the moment. Traditional HPC architectures
don't include GPUs. If you're a student or working on an unfunded project you're most likely working on either your laptop or an underpowered
lab machine. Google Colab provides free (as in beer, not speech) access to some pretty good GPU and TPU resources that are most likely better
than your laptop. The problem is you need to have data "local" to those instances to use them for free. For the most part, this is either
through example datasets or by uploading your data to Google drive. Not exactly the best dichotomy. Either get slow compute by keeping all of
your storage local or get slow/insufficient storage by having to upload huge datasets every time you make a tiny change to the preprocessing
of your data. This post is about connecting your storage to Google's compute in a way that scales pretty well for smaller (~10s of GB of data)
but not "small" projects.
