---
layout: post
title:  "Connecting Google Colab compute instances to your data"
date:   2021-08-18
categories: Software, Machine-learning
---

### Hey y'all: This is going to be a living document for the time being - I'm publishing this as I write, and so this won't be considered "finished" until I take this warning away

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

## Step 1: The storage side (Here, zarr)

For all of the dataset manipulations I'll assume use of `xarray`, since it's
mostly become the standard way to access Eath science data for computationally
intensive studies. Assuming we have a number of datasets we load them up like so

{% highlight python %}
    import xarray as xr
    mod_ds = xr.open_dataset('/path/to/model_output.nc')
    ref_ds = xr.open_dataset('/path/to/reference.nc')
{% endhighlight %}

Now suppose we want to run a classification task where we have "reference
categories"
such as those derived by the [US drought
monitor](https://droughtmonitor.unl.edu/)
that we want to train our ML model to predict. Given those have previously been
[one hot
encoded](https://machinelearningmastery.com/how-to-one-hot-encode-sequence-data-in-python/)
we similarly need a strategy to map our data from `model_output.nc` to such
encodings. We may also need to trim these modeled outputs to where we have
target/observed/truth data. These steps are what I will call the "Extract,
merge, transform" steps, which map well to the first two steps of the standard
extract, transform, load (ETL) workflow pattern.

For the sake of the post, let's assume we have gridded drought indicators which
span from $W^4-W^0$ (for wet-period--4 through 0), $N$ (for neutral), and
$D^0-D^4$ (for drought-period-0 through 4) which have been encoded into 11
categories in our `reference.nc` dataset(s).

## Step 2: The network interface (Here, fsspec)

Coming soon!

## Step 3: The compute service (Here, Google colab)

You guessed it - coming soon!

## Understanding performance, avoiding pitfalls

Gonna have to think on this before completion!

