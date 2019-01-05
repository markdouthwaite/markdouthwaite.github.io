---
title: "Introduction to Cython - Part 1: Getting Started"
layout: single
author_profile: true
date: 2019-01-05
tags: [Python, data science]
excerpt: "Python is slow. Use C or C++ if you need to do high-performance computing. That's the common wisdom. But Cython can let us write code in nearly pure Python and can yield huge speed boosts. It also lets us call C/C++ code directly from Python. This can be a huge productivity boost too."
share: true
---

## Introduction

Python is one of the most popular programming languages currently in use. It's often credited as being a highly productive language, user-friendly that is backed up by an extensive, mature and (generally) well maintained set of tools that can effectively service user demands across a wide range of fields. Python has enjoyed some popularity in scientific computing for quite a while now. This popularity has exploded in recent years with the adoption of Python as the de facto language of choice for a significant chunk of Artificial Intelligence (AI) research.

As positive as the general outlook for Python is at the moment, Python has some quite serious deficiencies in certain applications. Perhaps the most serious deficiency is the performance of most Python language implementations. Pure Python is often _horrendously_ slow. So slow, in fact, that many of the aforementioned scientific computing and AI tools are actually forced to wrap higher-performance code in languages such as C, C++ and Fortran in Python wrappers, and expose this code as user-friendly Python modules.

For Python programmers with little experience of these other languages, it might seem like a daunting task to port a Python implementation of some algorithm into C or C++ in order to get it to a point where their prototype implementation is fast enough to be useful for a given application. That's where Cython can come in _very_ useful. Cython is designed to make writing high-performance extensions for Python programs as simple as writing Python code itself.

## What is Cython?

Technically, Cython is a compiler for _both_ Python and the Cython programming language. The Cython programming language is a near-complete superset of Python itself. Practically, this means that almost all Python code is valid Cython code -- you can generally drop your Python code into the Cython compiler and have it work straight out-of-the-box. This raises a first important point: to use Cythonised code, you must compile your Cython modules. For those familiar with using interpreted languages, this process can seem a little odd, and sometimes can be quite confusing. We'll look at how to compile Cython modules briefly in this post, and in more detail in future posts, but for now it is enough to know that Cython takes your Cython module, uses it to generate some efficient C or C++ code and then compiles this into a binary that is accessible as Python module.

This general process is enhanced by a number of additional features provided by the Cython language. The most notable of these is the introduction of optional static type declarations. The Cython compiler can use these type declarations to dramatically improve the performance of compiled Cython code. Furthermore, Cython provides support for calling C/C++ code directly within a Cython module. This means we can integrate our Python application with pre-existing C/C++ code quite easily, or even call Python code from within C/C++ code. Finally, Cython also allows us to disable Python's Global Interpreter Lock (GIL), which can yield still greater performance boosts. Each of these capabilities will be touched on in future posts.

## Using Cython

Enough of my waffling, let's write some Cython. It's worth noting that I'm assuming the reader is familiar with Python and how to package and install their own Python modules. If you're not sure how to do this, there's a great guide [here](). The code for this post is available [here](), and has an accompanying notebook [here]() too. The examples use Python >= 3.5.

To start, we're going to look at a very simple calculation: computing the Fibonacci sequence up to an arbitrary value `n`. Here is a pure-Python function that does this:

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

Let's have a look at the `setup.py` script above. In our import statements, we need `from Cython.Build import cythonize`. The `cythonize` function compiles a list of Cython modules into C/C++ code. We also need `from setuptools import Extension`. This describes an extension module and everything needed to build it. The actual build process is managed by the setup script itself. 

A utility function `define_cython_extensions` has been provided to help configure the Cython extensions. More explanation of how to configure and build an extension will be given in future posts. For now, all that is required is that the `ext_modules` keyword argument is passed to the `setup` function with a list of extensions. Next, we need to build our extensions. In this case, in the root directory of the project we're going to run:

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

Where the `fibonacci.cpython-36m-darwin.so` binary will vary based upon your platform and Python version. To set our baseline, let's run the pure-Python code in `fibonacci_example.fibonacci`. We can run:

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

## Going Even Faster

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

An in-depth discussion of the modifications to the function have achieved is beyond the scope of this post, but fundamentally, the function has now been defined to take advantage of Cython's static type declaration capabilities. Adding this into the `fibonacci/fast/fibonacci.pyx` module and re-compiling the package, we can re-run:

```commandline
python3 -m timeit -r 10 -s 'from fibonacci_example.fast.fibonacci import fibonacci_faster' 'fibonacci_faster(1000000)'
```

Which gives me:

```
10000000 loops, best of 10: 0.0555 usec per loop
```

That's an almost 35x speed up over the pure Python code with surprisingly little effort!

## Conclusions

Hopefully this post has highlighted how straightforward it can be to get significant performance boosts to your standard Python code.
Accordign to the Cython documentation, Cython claims that these performance increases can be in the region of 60-90% for standard Python loops and control structures, and frequently between 100-1000x faster for numerical code. 

I've touched on a few of Cython's features in this post, and I'll be digging into more details on how to get your own code Cythonised efficiently in future posts. I'll also be giving some guidance on how you can use Cython to parallise your code effectively too. If you have any questions or suggestions for improvement, I'm always available via Twitter or through 

    'Premature optimization is the root of all evil' - D. Knuth

 a Cython compiled pybench runs more than 30% faster in total, while being 60-90% faster on control structures like if-elif-else and for-loops.

### A Quick Word of Warning 


