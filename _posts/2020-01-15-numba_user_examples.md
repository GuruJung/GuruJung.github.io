---
title: '[Numba 사용자 매뉴얼] 1.17 예제'
strapline: "Mandelbrot, Moving average, Multi-threading, ..."
header:
  overlay_image: /assets/images/index/python.jpg
categories:
  - "dev"
tag:
  - "numba"
  - "user_guide"
  - "translation"
  - "python"
classes: wide
toc: true
comments: true
mathjax: true
last_modified_at: 
---

Mandelbrot
==========

```python
#! /usr/bin/env python
# -*- coding: utf-8 -*-
from __future__ import print_function, division, absolute_import

from timeit import default_timer as timer
from matplotlib.pylab import imshow, jet, show, ion
import numpy as np

from numba import jit


@jit
def mandel(x, y, max_iters):
    """
    Given the real and imaginary parts of a complex number,
    determine if it is a candidate for membership in the Mandelbrot
    set given a fixed number of iterations.
    """
    i = 0
    c = complex(x,y)
    z = 0.0j
    for i in range(max_iters):
        z = z*z + c
        if (z.real*z.real + z.imag*z.imag) >= 4:
            return i

    return 255

@jit
def create_fractal(min_x, max_x, min_y, max_y, image, iters):
    height = image.shape[0]
    width = image.shape[1]

    pixel_size_x = (max_x - min_x) / width
    pixel_size_y = (max_y - min_y) / height
    for x in range(width):
        real = min_x + x * pixel_size_x
        for y in range(height):
            imag = min_y + y * pixel_size_y
            color = mandel(real, imag, iters)
            image[y, x] = color

    return image

image = np.zeros((500 * 2, 750 * 2), dtype=np.uint8)
s = timer()
create_fractal(-2.0, 1.0, -1.0, 1.0, image, 20)
e = timer()
print(e - s)
imshow(image)
#jet()
#ion()
show()
```

이동 평균 {#example-movemean}
==============

```python
#!/usr/bin/env python
"""
A moving average function using @guvectorize.
"""

import numpy as np

from numba import guvectorize

@guvectorize(['void(float64[:], intp[:], float64[:])'], '(n),()->(n)')
def move_mean(a, window_arr, out):
    window_width = window_arr[0]
    asum = 0.0
    count = 0
    for i in range(window_width):
        asum += a[i]
        count += 1
        out[i] = asum / count
    for i in range(window_width, len(a)):
        asum += a[i] - a[i - window_width]
        out[i] = asum / count

arr = np.arange(20, dtype=np.float64).reshape(2, 10)
print(arr)
print(move_mean(arr, 3))
```

멀티 쓰레딩
===============

쇼케이스 아래의 코드는 [nogil] 특징을 사용하여 잠재적인 성능 향상을 꾀할 수 있다.
예를 들어 4개의 코어 머신에서는 아래와 같은 결과를 출력하였다:

```
    numpy (1 thread)       145 ms
    numba (1 thread)       128 ms
    numba (4 threads)       35 ms
```

**Note:** 
파이썬3에서는 손으로 쓰레드를 만들어 작업을 디스패치하기 보다는
표준 [concurrent.futures](https://docs.python.org/3/library/concurrent.futures.html)
모듈을 사용하여 편리하게 수행할 수 있다.
{: .notice--warning}

```python
#!/usr/bin/env python
from __future__ import print_function, division, absolute_import

import math
import threading
from timeit import repeat

import numpy as np
from numba import jit

nthreads = 4
size = 10**6

def func_np(a, b):
    """
    Control function using Numpy.
    """
    return np.exp(2.1 * a + 3.2 * b)

@jit('void(double[:], double[:], double[:])', nopython=True, nogil=True)
def inner_func_nb(result, a, b):
    """
    Function under test.
    """
    for i in range(len(result)):
        result[i] = math.exp(2.1 * a[i] + 3.2 * b[i])

def timefunc(correct, s, func, *args, **kwargs):
    """
    Benchmark *func* and print out its runtime.
    """
    print(s.ljust(20), end=" ")
    # Make sure the function is compiled before we start the benchmark
    res = func(*args, **kwargs)
    if correct is not None:
        assert np.allclose(res, correct), (res, correct)
    # time it
    print('{:>5.0f} ms'.format(min(repeat(lambda: func(*args, **kwargs),
                                          number=5, repeat=2)) * 1000))
    return res

def make_singlethread(inner_func):
    """
    Run the given function inside a single thread.
    """
    def func(*args):
        length = len(args[0])
        result = np.empty(length, dtype=np.float64)
        inner_func(result, *args)
        return result
    return func

def make_multithread(inner_func, numthreads):
    """
    Run the given function inside *numthreads* threads, splitting its
    arguments into equal-sized chunks.
    """
    def func_mt(*args):
        length = len(args[0])
        result = np.empty(length, dtype=np.float64)
        args = (result,) + args
        chunklen = (length + numthreads - 1) // numthreads
        # Create argument tuples for each input chunk
        chunks = [[arg[i * chunklen:(i + 1) * chunklen] for arg in args]
                  for i in range(numthreads)]
        # Spawn one thread per chunk
        threads = [threading.Thread(target=inner_func, args=chunk)
                   for chunk in chunks]
        for thread in threads:
            thread.start()
        for thread in threads:
            thread.join()
        return result
    return func_mt


func_nb = make_singlethread(inner_func_nb)
func_nb_mt = make_multithread(inner_func_nb, nthreads)

a = np.random.rand(size)
b = np.random.rand(size)

correct = timefunc(None, "numpy (1 thread)", func_np, a, b)
timefunc(correct, "numba (1 thread)", func_nb, a, b)
timefunc(correct, "numba (%d threads)" % nthreads, func_nb_mt, a, b)
```

[nogil]: /dev/numba_user_jit#jit-nogil
