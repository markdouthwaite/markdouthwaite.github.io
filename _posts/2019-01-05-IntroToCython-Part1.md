---
title: "Introduction to Cython - Part 1: Getting Started"
layout: single
author_profile: true
date: 2019-08-06
tags: [Python, data science]
excerpt: "This post gives a quick introduction on how to accelerate your Python code using Cython."
share: true

header:
    teaser: /images/Cython-logo.svg

---

<div style="text-align:center"><img src="/images/Cython-logo.svg" /></div>

This post looks at how you can go about getting some pretty impressive speed boosts in your Python code with surprisingly little effort with Cython, a programming language and compiler that is designed to enable straightforward interoperability between Python, C/C++ and the eponymous Cython language.

## What is Cython?

If you've worked in Python's scientific programming world for any length of time, chances are you will have at least heard of Cython. It's used in one way or another in packages ranging from the ubiquitous NumPy, SciPy and Scikit-Learn packages, through to the likes of [Implicit]() and [LightFM](), and even [TensorFlow]().

Cython is a programming language and compiler. As a programming language, Cython is a near-complete superset of Python: almost all Python code is valid Cython code, and can therefore be compiled with the Cython compiler. Consequently, you can simply drop your Python code into the Cython compiler and have it generate some nicely optimised C binaries that you can access from your other Python code. That's what this post looks at: what sort of performance boosts can we get by dropping a simple calculation into Cython?

## Time to Cythonise

To answer this, let's write some Cython. It's worth noting that I'm assuming the reader is familiar with Python and how to package and install their own Python modules. If you're not sure how to do this, there's a good guide from the Python packaging site [here](https://python-packaging.readthedocs.io/en/latest/minimal.html). The code for this post is available [here](https://github.com/markdouthwaite/cython-examples). The examples require Python >= 3.5.

To start, we're going to look at a very simple calculation -- computing the Fibonacci sequence up to an arbitrary value of `n`. This is the canonical Cython example. Here is a pure-Python function that does this:

```python
# fibonacci_example/fibonacci.py
def fibonacci(n):
    a, b = 0, 1
    while b < n:
        a, b = b, a + b
    return b
```

Let's assume this is some computation we've identified as being a bottleneck for our application. To begin, our minimal Python package might have a structure like:

```
+-- fibonacci_example
|   +-- __init__.py
|   +-- fibonacci.py
+-- setup.py
```

A first step can be to drop this function into a Cython module. Cython source files are denoted with the `.pyx` extension. Let's update our package structure to look like:

```
+-- fibonacci_example
|   +-- __init__.py
|   +-- fibonacci.py
|   +-- fast
|       +-- __init__.py
|       +-- fibonacci.pyx
+-- setup.py
```

Inside the `fibonacci.pyx` file, we can simply copy our original function, so our new file looks like:

```python
# fibonacci_example/fast/fibonacci.pyx
def fibonacci(n):
    a, b = 0, 1
    while b < n:
        a, b = b, a + b
    return b
```

Pure Python! Next, we need to compile our new chunk of Cython code. To do this, we need to define our Cython extensions and Python's setup tools will handle the compilation step. Our `setup.py` file looks like:

```python
import os
from Cython.Build import cythonize
from setuptools import Extension, setup, find_packages


def os_path(import_path: str, ext: str) -> str:
    """
    Build the path to a module from it's import path.
    """

    return os.path.join(*import_path.split("."))+ext


def define_cython_extensions(*extensions: str,
                             link_args: list=None,
                             compile_args: list=None,
                             language: str="c",
                             file_ext: str=".pyx") -> list:
    """
    Build a list of Cython extension modules.
    """

    modules = []
    for extension in extensions:
        modules.append(Extension(extension,
                       [os_path(extension, file_ext)],
                       language=language,
                       extra_compile_args=compile_args,
                       extra_link_args=link_args,
                       include_dirs=["."]))

    cythonize(modules)

    return modules


# list your extensions here
ext_modules = define_cython_extensions(
    "fibonacci_example.fast.fibonacci",
    # more extensions here!
)

setup(name="fibonacci-example",
      version="1.0.0",
      license="MIT",
      packages=find_packages(exclude=["contrib", "docs", "tests*"]),
      ext_modules=ext_modules,
      setup_requires=["Cython>=0.24"]
    )
```

You'll need to install `Cython` if you don't already have it installed. You can do this with: `pip install cython`. 

Let's have a look at the `setup.py` script above. In our import statements, we need `from Cython.Build import cythonize`. The `cythonize` function compiles a list of Cython modules into C/C++ code. We also need `from setuptools import Extension`. This describes an extension module and everything needed to build it. By default, the build process is managed by the setup script itself.

I've added a utility function `define_cython_extensions` to help configure the Cython extensions. More explanation of how to configure and build an extension will be given in future posts. For now, all that is required is that the `ext_modules` keyword argument is passed to the `setup` function with a list of extensions. Next, we need to build our extensions. In this case, in the root directory of the project we're going to run:

```commandline
python setup.py build_ext --inplace
```

You should now see that your package structure looks similar to:

```
+-- fibonacci_example
|   +-- __init__.py
|   +-- fibonacci.py
|   +-- fast
|       +-- __init__.py
|       +-- fibonacci.c
|       +-- fibonacci.cpython-36m-darwin.so
|       +-- fibonacci.pyx
+-- setup.py
```

Where the `fibonacci.cpython-36m-darwin.so` binary will vary based upon your platform and Python version. 

## Benchmarking

Now to testing. To set our baseline benchmark, let's run the pure-Python code in `fibonacci_example.fibonacci`. We can run:

```commandline
python3 -m timeit -r 10 -s 'from fibonacci_example.fibonacci import fibonacci' 'fibonacci(1000000)'
```

On my machine, my results are:

`1000000 loops, best of 10: 1.89 usec per loop`

Now let's look at our new Cythonised version of the `fibonacci` function:

```commandline
python3 -m timeit -r 10 -s 'from fibonacci_example.fast.fibonacci import fibonacci' 'fibonacci(1000000)'
```

Again, my results are:

`1000000 loops, best of 10: 0.923 usec per loop`

That's more than twice as fast for no changes to my code!

## More Acceleration

Twice as fast is great, but we can definitely do better. The code snippet below shows the `fibonacci` function modified to use some Cython language features:

```python
cpdef int fibonacci_faster(int n):
    """
    Print the Fibonacci series up to n.
    """
    cdef int a = 0
    cdef int b = 1
    while b < n:
        a = b
        b = a + b
    return b
```

Here we're using Cython's static type definitions. We won't go into detail on the syntax and variants of these here, but fundamentally, this will give us some additional speed boosts! Adding this into the `fibonacci/fast/fibonacci.pyx` module and re-compiling the package, we can re-run:

```commandline
python3 -m timeit -r 10 -s 'from fibonacci_example.fast.fibonacci import fibonacci_faster' 'fibonacci_faster(1000000)'
```

Which gives me:

```
10000000 loops, best of 10: 0.0555 usec per loop
```

That's an almost 35x speed up over the pure Python code with surprisingly little effort! We'll look into the details of this snippet in the next post.

## Conclusions

Hopefully this post has highlighted how straightforward it can be to get significant performance boosts to your standard Python code.
Accordign to the Cython documentation, Cython claims that these performance increases can be in the region of 60-90% for standard Python loops and control structures, and frequently between 100-1000x faster for numerical code.

I've touched on a few of Cython's features in this post, and I'll be digging into more details on how to get your own code Cythonised efficiently in future posts. I'll also be giving some guidance on how you can use Cython to parallise your code effectively too. If you have any questions or suggestions for improvement, I'm always available via Twitter or through my [website](mark.douthwaite.io).

And just one more quick word: Cython is very powerful, but always remember the valuable advice from Donald Knuth:

    'Premature optimization is the root of all evil' - D. Knuth

So don't go spending hours trying to optimise your code until you've arrived at a good solution, and identified concrete benefits from doing this. But when you reach the point you _must_ optimise, Cython could be a good bet!
