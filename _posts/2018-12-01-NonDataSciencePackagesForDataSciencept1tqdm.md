---
title: "Python Utilities for Data Scientists - Part 1: tqdm"
layout: single
author_profile: true
date: 2018-12-01 
tags: [data science, python]
excerpt: "Data Scientists can sometimes make life difficult for themselves simply because the focus of their work is relatively narrow in comparison to the capabilities of a language like Python. This series of posts looks at general-purpose packages that may save Data Scientists a fair bit of time."
share: true
---

This series of posts is going to give an overview of some interesting, useful and/or time-saving general-purpose packages that can make the lives of Data Scientists a little easier. I thought I'd put this together after finding a few-to-many DS repositories, scripts and notebooks that made life far too difficult for themselves.

## TQDM

To kick things off, let's look at `tqdm`, a fantastic, easy-to-use, extensible progress bar package. It makes adding fast, informative progress bars to Python processes extremely easy. If you're a Data Scientist or Machine Learning (ML) Engineer with any degree of experience, you'll no doubt have used or developed algorithms or data transformations that can take many hours to complete.

Invariably, many Data Scientists opt to simply print status messages to console, or in some slightly more sophisticated cases use the (excellent and recommended) built-in `logging` module. In a lot of cases this is fine. However, if you're running a task with many hundreds of steps, or over a data structure with many millions of elements, these approaches are sometimes a little unclear and gratuitous, and frankly kind of ugly.

<figure><img src="http://mark.douthwaite.io/wp-content/uploads/2018/11/Screenshot-2018-11-26-at-22.20.37-1024x104.png" sizes="(max-width: 1024px) 100vw, 1024px" srcset="http://mark.douthwaite.io/wp-content/uploads/2018/11/Screenshot-2018-11-26-at-22.20.37-1024x104.png 1024w, http://mark.douthwaite.io/wp-content/uploads/2018/11/Screenshot-2018-11-26-at-22.20.37-300x31.png 300w, http://mark.douthwaite.io/wp-content/uploads/2018/11/Screenshot-2018-11-26-at-22.20.37-768x78.png 768w, http://mark.douthwaite.io/wp-content/uploads/2018/11/Screenshot-2018-11-26-at-22.20.37.png 1080w" alt="" width="1024" height="104" /><figcaption>Example output from <code class="prettyprint">tqdm</code></figcaption></figure>

That's where `tqdm` can come in. It has a nice clean API that lets us quickly add progress bars to our code. Plus it has a lightweight 'time-remaining' estimation algorithm built in to the progress bar too. When I first came across `tqdm` (while exploring the `implicit` library), my mind immediately went to how I could use it to make my own ML algorithm outputs a lot tidier. Take a look at the example below:

```python
from tqdm import tqdm
k = 100
with tqdm(desc="Training Epoch", total=k) as progress:
    for epoch in range(k):
        progress.update(1)  # increments the progress bar
```

In this simple example, we set up a `tqdm` progress bar that expects a process of 100 steps. We then run our mock training loop, each time updating the progress bar when the step is completed. We can also update the progress bar by arbitrary amounts if we break out of the loop too. That's two lines of code (plus the import statement) to get a rich progress bar in your code.

## Pandas Integration

Beyond cool little additions to your program's outputs,`tqdm`</code>`also integrates nicely with other widely used packages. Probably the most interesting integration for Data Scientists is with Pandas, the ubiquitous Python data analysis library. Take a look at the example below:

```python
df = pd.read_csv("weather.csv")
tqdm.pandas(desc="Applying Transformation")
df.progress_apply(lambda x: x)
```

This would give us:

<figure><img src="http://mark.douthwaite.io/wp-content/uploads/2018/11/Screenshot-2018-12-01-at-12.41.27.png" sizes="(max-width: 1024px) 100vw, 1024px" alt="" width="1024" height="104" /><figcaption>Example output from `tqdm`</figcaption></figure>

Technically, the `tqdm.pandas` method monkey patches the `progress_apply` method onto Pandas data structures, giving them a modified version of the commonly used `apply` method. Practically, when we call the `progress_apply` method, the package wraps the standard Pandas 'apply' method with a tqdm progress bar. This can come in really handy when you're processing large data frames!
