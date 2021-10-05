---
layout: post
title:  "Connecting Google Colab compute instances to your data
date:   2021-08-18
categories: Software, Machine-learning
---

### Hey y'all: This is going to be a living document for the time being - I'm
publishing this as I write, and so this won't be considered "finished" until I
take this warning away

It's pretty indisputable that machine learning is both trendy and effective in
a number of Earth & environmental
sciences. With the speed and uptake of, particularly, deep-learning methods
it's hard to keep up with best practices
for training and deploying models. It's also expensive to roll your own
hardware to build a DL platform for these applications,
where data sizes can be large, heterogeneous, and generally difficult to
wrangle. That is to say, the data space of Earth science
is moving as fast as machine-learning. In well funded projects this isn't
necessarily a problem, but for exploring hypotheses and
proofs-of-concept it's important to have access to hardware that can be useful
on limited timelines. Particularly, this can be a
daunting landscape for students who are expected to learn the science and
technology concurrently.

Okay, passive voice aside, this is a tough space to operate in. GPUs are
absurdly expensive at the moment. Traditional HPC architectures
don't include GPUs. If you're a student or working on an unfunded project
you're most likely working on either your laptop or an underpowered
lab machine. Google Colab provides free (as in beer, not speech) access to some
pretty good GPU and TPU resources that are most likely better
than your laptop. The problem is you need to have data "local" to those
instances to use them for free. For the most part, this is either
through example datasets or by uploading your data to Google drive. Not exactly
the best dichotomy. Either get slow compute by keeping all of
your storage local or get slow/insufficient storage by having to upload huge
datasets every time you make a tiny change to the preprocessing
of your data. This post is about connecting your storage to Google's compute in
a way that scales pretty well for smaller (~10s of GB of data)
but not "small" projects.


## The conceptual diagram

Before getting into details and code, I want to take a second to go over all
the pieces that this post constructs.
This conceptual diagram is shown below, in figure 1. The high level
architecture here shows that there are two machines, one for storage and data
manipulation as well as one for managing compute at training time.


<figure markdown='1'>
  ![](../../../../../../imgs/async_ml_training_diagram.png)
  <figcaption>
  Figure 1: The conceptual diagram outlining the workflow that motivates this
  post
  </figcaption>
</figure>

On the storage side (left portion of figure 1) the raw datasets are loaded,
subsetted, transformed, and saved out into the intermediate format that will be
used to train the machine learning model. Ideally this involves transformation
into a data structure and
file format that supports chunking and must be able to be loaded over network
interfaces.
In this post I will focus on using zarr, which not only supports both but is
almost perfectly
designed for such a workflow. The need for network access should be obvious
here, given that the network is the mediator between the storage machine and
compute service, but the need for "chunking" may not. To get a sense of how
chunking can affect data latency, you can look back at my previous [post on
chunking NetCDF data.] Though similar, in that post I was interested in
optimizing writing output, while in this one I am more interested in reading
input. In both cases, having a good chunking strategy is key, and further the
chunking strategy we use here will be crucial to how we do mini-batch training
on the compute service.

This manifests itself on the right hand side of figure 1, in the "Data loader"
which is responsible for loading these minibatches, which will be consumed by
the training strategy. Along with an ML model definition, this training
strategy is iterated to optimize the best model, given the training data
provided by the data loader. The output of this workflow is the trained model
which then can be used in forward mode for whatever purpose it was designed
for.

