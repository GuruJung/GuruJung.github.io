---
title: '[Numba 사용자 매뉴얼] 1.1 5분 가이드'
strapline: "Numba는 NumPy 배열과 함수 및 루프를 사용하는 파이썬 코드에 가장 잘 작동하는 즉석 컴파일러이다."
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

Numba는 NumPy 배열과 함수 및 루프를 사용하는 파이썬 코드에 가장 잘 작동하는 즉석 컴파일러이다.
Numba에서 공통적으로 사용하는 방법은 당신의 함수에 일련의 데코레이터를 적용하는 것이다.
그렇게 함으로써 Numba는 당신의 코드를 컴파일 할 수 있다.
Numba로 데코레이트된 함수에 대한 호출이 있을 때 해당 함수는 \"즉석\"에서 바로 기계어 코드로 컴파일되고 원래의 기계어 코드 속도로 실행된다!

먼저 Numba는 다음과 같은 환경에서 작동한다:

-   OS: Windows (32/64 비트), OSX and Linux (32/64 비트)
-   Architecture: x86, x86\_64, ppc64le. armv7l, armv8l (aarch64)에 대해서는 실험 버전.
-   GPUs: Nvidia CUDA. AMD ROC에 대해서는 실험 버전.
-   CPython
-   NumPy 1.10 버전부터 지원.

## Numba 구하기

Numba는 [아나콘다 파이썬 배포본](https://www.anaconda.com/)에서 [콘다](https://conda.io/docs/) 패키지로 구할 수 있다.

    $ conda install numba

또는 Numba를 pip으로 설치할 수 있다.

    $ pip install numba

Numba를 [소스에서 컴파일](/dev/numba_user_installing#installing-from-source)할 수도 있지만, 
Numba는 핵심 패키지로 사용될 수 있도록 가능한 한 최소한도의 의존성만 있도록 유지되고 있긴 하나,
아래와 같은 여분의 패키지를 설치함으로써 부가 기능을 더 제공한다:

-   `scipy` - `numpy.linalg` 함수를 컴파일할 수 있도록 한다.
-   `colorama` - 역추적/에러 메시지를 컬러풀하게 강조할 수 있도록 한다.
-   `pyyaml` - YAML 설정 파일을 통해서도 Numba 설정을 할 수 있도록 한다.
-   `icc_rt` - Numba가 인텔 SVML (x86\_64용 고성능 단-벡터 수학 라이브러리)를 사용할 수 있도록 해준다. [성능 향상 팁](/translation/numba_user_performance-tips)에 설치 방법이 있다.

## Numba가 내 코드에 작동할 수 있을까?

그에 대한 답은 당신의 코드가 어떻게 생겼지는에 따라 다르다. 
당신의 코드가 수학을 많이 쓰는, 수치 지향적이거나 NumPy를 많이 쓰거나 또는 루프를 많이 쓰고 있다면,
Numba를 쓰는 데 적당하다.
이런 예들의 함수에 가장 기본적인 Numba의 JIT 데코레이터 `@jit`을 적용했을 때
잘 작동하는 예와 작동하지 않는 예를 보자.

아래처럼 보이는 코드에 대해서 Numba는 잘 작동한다:

```python
    from numba import jit
    import numpy as np

    x = np.arange(100).reshape(10, 10)

    @jit(nopython=True) # Set "nopython" mode for best performance, equivalent to @njit
    def go_fast(a): # Function is compiled to machine code when called the first time
        trace = 0
        for i in range(a.shape[0]):   # Numba likes loops
            trace += np.tanh(a[i, i]) # Numba likes NumPy functions
        return a + trace              # Numba likes NumPy broadcasting

    print(go_fast(x))
```

아래와 같은 코드에 대해서는 Numba가 잘 작동하지 않는다:

```python
    from numba import jit
    import pandas as pd

    x = {'a': [1, 2, 3], 'b': [20, 30, 40]}

    @jit
    def use_pandas(a): # Function will not benefit from Numba jit
        df = pd.DataFrame.from_dict(a) # Numba doesn't know about pd.DataFrame
        df += 1                        # Numba doesn't understand what this is
        return df.cov()                # or this!

    print(use_pandas(x))
```

Numba는 Pandas를 잘 모르기 때문에 그 결과 Numba는 본 코드를 단순히 해석기를 통해 실행될 뿐만 아니라
오히려 Numba 내부의 오버헤드가 부가됨을 주목하라!

## `nopython` 모드

Numba `@jit` 데코레이터는 기본적으로 `nopython`, `object` 컴파일 모드로 작동한다.
위의 `go_fast` 예제에서는 `@jit` 데코레이터 안에 `nopython=True`가 세팅되는데,
이것은 Numba가 항상 `nopython` 모드로 작동되도록 한다.
`nopython` 컴파일 모드의 행동은 데코레이트되는 함수을 필수적으로 컴파일함으로써 파이썬 해석기의 개입 없이 실행시키는 것이다.
본 방법은 항상 최고의 성능으로 이끌기 때문에 추천되는 방법이다.

`nopython` 모드에서 컴파일이 실패한다면 Numba는 `object` 모드로 컴파일을 시도한다. 
이것은 위의 `use_pandas` 예제처럼 `nopython=True`가 세팅되어 있지 않은 경우에 작동하는,
`@jit` 데코레이터의 안전 모드이다.  
이 모드에서는 Numba가 컴파일 가능한, 루프 등의 코드를 확인하고 그것들을 기계어 코드로 실행할 수 있는 함수로 컴파일한다.
그리고 나머지 코드에 대해서는 해석기를 통해 실행한다.
최적의 성능을 위해서는 이런 모드를 사용하는 것을 피해야 한다!

## Numba의 성능을 측정하는 방법

먼저, 어떤 함수의 기계어 코드 버전을 실행하기 전에 주어진 인수 타입으로 당신의 함수를 Numba가 컴파일해야 한다는 것을 상기하자.
이 일은 시간이 걸리지만, 컴파일이 한번 완료되면 Numba는 해당 기계어 코드 버전을 저장해 놓는다.
그리고 같은 타입으로 다시 호출된다면 다시 컴파일하지 않고 미리 저장된 버전을 재사용한다.

성능을 측정할 때 자주하는 실수는 위의 행동을 고려하지 않고 함수를 컴파일하는데 걸리는 시간을 포함하여 한번만 코드를 실행하고 시간을 재는 것이다.

예를 들어:

```python
    from numba import jit
    import numpy as np
    import time

    x = np.arange(100).reshape(10, 10)

    @jit(nopython=True)
    def go_fast(a): # Function is compiled and runs in machine code
        trace = 0
        for i in range(a.shape[0]):
            trace += np.tanh(a[i, i])
        return a + trace

    # DO NOT REPORT THIS... COMPILATION TIME IS INCLUDED IN THE EXECUTION TIME!
    start = time.time()
    go_fast(x)
    end = time.time()
    print("Elapsed (with compilation) = %s" % (end - start))

    # NOW THE FUNCTION IS COMPILED, RE-TIME IT EXECUTING FROM CACHE
    start = time.time()
    go_fast(x)
    end = time.time()
    print("Elapsed (after compilation) = %s" % (end - start))
```

위 코드는, 예를 들면, 아래와 같은 결과를 출력한다:

    Elapsed (with compilation) = 0.33030009269714355
    Elapsed (after compilation) = 6.67572021484375e-06

Numba JIT이 어떤 코드에 가하는 영향을 정확히 측정하려면, [timeit](https://docs.python.org/3/library/timeit.html) 모듈 함수를 사용하여 실행 시간을 재야 한다.
이 방법은 여러 번 반복해서 실행하기 때문에 처음 실행시 행해졌던 컴파일 시간을 적절히 고려한다.

부가적으로 컴파일 시간이 문제가 된다면, Numba JIT은 컴파일된 함수의 [on-disk caching](/dev/numba_user_jit)을 지원하고
또한 [Ahead-Of-Time](/dev/numba_user_pycc) 컴파일 모드도 지원한다.

## 얼마나 빠른가?

Numba가 `nopython` 모드로 작동하거나 또는 최소한 어떤 루프가 있어서 컴파일한다든가 하면 Numba는 현재의 특정 CPU에 맞는 타켓 컴파일을 수행한다.
속도 향상은 응용프로그램에 따라 달라지지만 대략 백배이상의 성능 향상이 있을 수 있다.
[성능 가이드](/dev/numba_user_performance-tips)는 여분의 성능 향상을 얻기 위한 여러 옵션을 다루고 있다.  

## Numba는 어떻게 작동하는가?

Numba는 데코레이트된 함수의 파이썬 바이트 코드를 읽어서 해당 함수의 입력 인수의 타입에 대한 정보와 결합한다.
그리고 그것들을 분석하고 최적화한 후 마지막으로 LLVM 컴파일러 라이브러리를 사용하여 현재의 CPU 능력을 고려하여 해당 함수의 기계어 코드 버전을 생성한다.
이렇게 하여 컴파일된 버전은 해당 함수가 호출될 때마다 사용된다.

## 다른 관심사

Numba는 몇 가지 데코레이터를 가지고 있는데 지금까지는 `@jit`만을 봐 왔다.
아래에는 다른 것들도 보여준다:

-   `@njit` - 이것은 `@jit(nopython=True)`에 대한 별칭이며, 매우 자주 쓰이는 데코레이터이다.
-   `@vectorize` - NumPy `ufunc`을 생성한다 (모든 `ufunc` 메쏘드가 지원된다). [여기](/dev/numba_user_vectorize)에 문서가 있다.
-   `@guvectorize` - NumPy 일반화된 `ufunc`을 생성한다. [여기](/dev/numba_user_vectorize)에 문서가 있다.
-   `@stencil` - 연산과 같은 스텐실을 위한 커널로서의 함수를 선언한다. [여기](/dev/numba_user_stencil)에 문서가 있다.
-   `@jitclass` - jit을 알고 있는 클래스 생성용. [여기](/dev/numba_user_jitclass)에 문서가 있다.
-   `@cfunc` - (C/C++로부터 호출될 수 있는) 네이티브 콜백으로 사용될 함수를 선언한다. [여기](/dev/numba_user_cfunc)에 문서가 있다.
-   `@overload` - `@overload(scipy.special.j0)`와 같은 예처럼, 기존의 어떤 함수를 nopython 모드로 사용하기 위해서 해당 함수에 대해서 별도의 구현물을 등록한다. [여기](http://numba.pydata.org/numba-doc/latest/extending/high-level.html#high-level-extending)에 문서가 있다.

일부 데코레이터에서 쓸 수 있는 여분의 옵션들:

-   `parallel = True` - 함수 코드의 [자동 병렬화](numba_user_parallel.html)를 활성화한다.
-   `fastmath = True` - fast-math 기능을 활성화한다.

ctypes/cffi/cython 상호운용성:

-   `cffi` - `nopython` 모드에서 CFFI 함수를 호출하는 것이 가능하다.
-   `ctypes` - `nopython` 모드에서 ctypes으로 싸여진 함수를 호출하는 것이 가능하다. 
-   Cython에 의해서 익스포트된 함수도 호출 가능하다.

## GPU 타겟

Numba는 [Nvidia CUDA](https://developer.nvidia.com/cuda-zone)와 (실험적으로) [AMD ROC](https://rocm.github.io/) GPU를 지원한다.
순수 파이썬으로 커널만 작성하기만 하면 대신 Numba가 계산과 데이터를 핸들링한다 (또는 당신이 직접적으로 이것을 핸들링할 수도 있다).
[CUDA](http://numba.pydata.org/numba-doc/latest/cuda/index.html#cuda-index)와 [ROC](http://numba.pydata.org/numba-doc/latest/roc/index.html#roc-index)에 Numba 관련 문서가 있다.

**Note:** 
이 글은 Numba user manual을 번역한 글입니다.
[목차](/dev/numba_user_index)를 보려면 [여기](/dev/numba_user_index)를 클릭하세요.
{: .notice--warning}
