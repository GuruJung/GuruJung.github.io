---
title: '[Numba 사용자 매뉴얼] 1.13 성능향상 팁'
strapline: "본 문서는 코드로부터 최상의 성능을 얻기 위한 가이드이다."
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

본 문서는 코드로부터 최상의 성능을 얻기 위한 가이드이다.
두 예제가 사용되는데, 동기 부여를 위한 순수한 교육적 이유로 고안되었다.
첫번째 예제는 삼각 방정식 `cos(x)^2 + sin(x)^2` 계산이고,
두번째는 간단한 벡터의 element-wise 루트 근의 합계이다.
모든 성능 수치는 직접적이며, 따로 언급이 되지 않는다면 `np.arange(1.e7)` 입력에 대해
인텔 `i7-4790` CPU (4 hardware threads)를 사용하였다.

**Note:** 
고성능 코드를 얻기 위한 효과적인 접근법은 실제 데이터로 코드를 프로파일링하고
성능 튜닝하는 것이다.
여기에서 제시되는 정보는 기능을 보여주기 위함이지 표준 지침으로 행하라는 것은 아니다.
{: .notice--warning}

no python 모드와 object 모드
=============================

`@jit`는 Numba가 제공하는 가장 유연한 데코레이터여서 많이 쓰인다.
특히, `@jit`는 항상 두 가지 모드의 컴파일을 제공한다;
그것은 데코레이트되는 함수를 우선 no python 모드로 컴파일을 시도하며,
만약 컴파일에 실패한다면, object 모드에서 컴파일을 시도한다.
object 모드에서 루프 리프팅 (looplifting)과 같은 기능의 사용하여 어느 정도 성능 향상이 있겠으나,
no python 모드에서 함수를 컴파일하는 것이 성능 향상을 가장 크게 가져온다.
그래서 no python 모드로만 컴파일 시도를 하고, 만약 컴파일에 실패한다면
예외를 주도록 하는 방식을 적용하기 위해서는 
`@njit` 또는 `@jit(nopython=True)` 데코레이터를 사용하면 된다
(`@njit`은 `@jit(nopython=True)`의 별칭이다).


루프
=====

Numpy는 벡터 연산을 주로 사용하는 경우를 가정하고 개발되었는데, Numba 역시 루프를 잘 다룬다.
C나 포트란에 친숙한 사용자들의 경우 이런 스타일로 파일썬을 작성하는 것은 Numba에서 문제가 되지 않는다
(C 언어를 LLVM으로 많이 컴파일한다).
예를 들면:

```python
    @njit
    def ident_np(x):
        return np.cos(x) ** 2 + np.sin(x) ** 2

    @njit
    def ident_loops(x):
        r = np.empty_like(x)
        n = len(x)
        for i in range(n):
            r[i] = np.cos(x[i]) ** 2 + np.sin(x[i]) ** 2
        return r
```

 `@njit` 데코레이션이 없을 때는 vectorized 함수(ident_np)가 몇십~몇백배 더 빠르게 실행되지만,
 `@njit`으로 데코레이션을 하면, 위 두 코드 모두 거의 동일한 속도를 보여준다.

+-----------------+-------+----------------+
| Function Name   | \@njit | Execution time |
+=================+=======+================+
| `ident_np`      | No    | > 0.581s       |
+-----------------+-------+----------------+
| `ident_np`      | Yes   | > 0.659s       |
+-----------------+-------+----------------+
| `ident_loops`   | No    | > 25.2s        |
+-----------------+-------+----------------+
| `ident_loops`   | Yes   | > 0.670s       |
+-----------------+-------+----------------+

Fastmath 
========

IEEE 754 준수를 엄격히 할 필요가 없는 응용 프로그램의 경우에 
수치 제약을 해제함으로써 부가적인 성능 향상을 얻을 수 있다.
Numba에서 이런 제약을 해제하기 위해서는 `fastmath` 키워드를 사용하면 된다:

```python
    @njit(fastmath=False)
    def do_sum(A):
        acc = 0.
        # without fastmath, this loop must accumulate in strict order
        for x in A:
            acc += np.sqrt(x)
        return acc

    @njit(fastmath=True)
    def do_sum_fast(A):
        acc = 0.
        # with fastmath, the reduction can be vectorized as floating point
        # reassociation is permitted.
        for x in A:
            acc += np.sqrt(x)
        return acc
```

+-----------------+-----------------+
| Function Name   | Execution time  |
+=================+=================+
| `do_sum`        | > 35.2 ms       |
+-----------------+-----------------+
| `do_sum_fast`   | > 17.8 ms       |
+-----------------+-----------------+

Parallel=True
=============

코드가 병렬화가 [numba-parallel-supported]을 포함하고 있다면 Numba는 GIL 적용 없이
다중의 네이티브 쓰레드상에서 병렬로 실행되는 버전을 컴파일할 수 있다.
`parallel` 키워드를 추가함으로써 이런 병렬성은 자동으로 수행된다.

```python
    @njit(parallel=True)
    def ident_parallel(A):
        return np.cos(x) ** 2 + np.sin(x) ** 2
```

실행 시간은 다음과 같다:

  --------------------------------------
  Function Name        Execution time
  -------------------- -----------------
  `ident_parallel`     112 ms
  --------------------------------------

`parallel=True`을 가진 함수의 실행 속도는 NumPy보다 5배 빠르고, 표준 `@njit`보다는 6배 더 빨랐다.

Numba는 OpenMP처럼 명시적 병렬 루프도 지원한다. 
`numba.prange`를 사용하여 루프가 병렬로 실행되어야 함을 알려준다.
이 함수는 파이썬 `range`처럼 행동하는데 실제로 `parallel=True`가 세팅되어 있지 않으면
정말로 `range`와 동일하다.
`prange`가 쓰인 루프는 병렬 계산뿐만 아니라 reduction도 가능하다.

합을 통한 축소(reduction) 예제를 시작하기 전에, 순서에 상관없이 누적을 하여도 안전해야 함을 유의해야 한다.
`n`번 반복 루프는 `prange`를 사용하여 병렬화된다.
병렬 실행의 경우 비순차적 실행(out of order execution)이
이미 가정되고 있기 때문에 `fastmath=True` 키워드를 쉽게 추가할 수 있다
(각 쓰레드는 부분 합을 계산한다):

```python
    @njit(parallel=True)
    def do_sum_parallel(A):
        # each thread can accumulate its own partial sum, and then a cross
        # thread reduction is performed to obtain the result to return
        n = len(A)
        acc = 0.
        for i in prange(n):
            acc += np.sqrt(A[i])
        return acc

    @njit(parallel=True, fastmath=True)
    def do_sum_parallel_fast(A):
        n = len(A)
        acc = 0.
        for i in prange(n):
            acc += np.sqrt(A[i])
        return acc
```

실행 시간은 아래와 같은데, `fastmath`를 사용하면 한번 더 성능을 향상시킬 수 있음을 알 수 있다.
Execution times are as follows, `fastmath` again improves performance.

+-------------------------+-----------------+
| Function Name           | Execution time  |
+=========================+=================+
| `do_sum_parallel`       | > 9.81 ms       |
+-------------------------+-----------------+
| `do_sum_parallel_fast`  | > 5.37 ms       |
+-------------------------+-----------------+

Intel SVML
==========

인텔은 단벡터 수학 라이브러리(SVML: Short Vector Math Library)를 제공하고 있는데,
그것은 컴파일러 내부 사용 용도의 대규모의 최적화된 함수들을 포함하고 있다.
`icc_rt` 패키지가 환경에 보인다면 (또는 SVML 라이브러리가 간단하게 발견될 수 있다면) 
Numba는 LLVM 백엔드를 설정하여 자동으로 가능하면 SVML 내부 함수를 사용한다.
SVML은 각 함수에 대해서 고정밀 또는 저정밀 버전을 제공하는데, `fastmath` 키워드를 통해 결정되는 버전도 또한 제공한다.
기본 옵션은 `1 ULP`만큼 정확한 고정밀 버전을 사용하는 것인데,
`fastmath`가 `True`로 설정되면 저정밀 버전 (`4 ULP`내의 정확도)을 사용한다.

우선 SVML을 설치하려면, conda를 사용한 예제의 경우:

    conda install -c numba icc_rt

위의 함수 예제 `ident_np`를 `@njit` 및 SVML 사용 여부 등의 여러 조합을 사용하여 실행한 결과는
아래와 같다 (입력 크기는 `np.arange(1.e8)`).
참고로 단순히 NumPy 함수를 실행한 경우는 `5.84s`였다:

  ----------------------------------------------------------------
  `@njit` kwargs                      SVML     Execution time
  ----------------------------------- -------- -------------------
  `None`                              No       5.95s

  `None`                              Yes      2.26s

  `fastmath=True`                     No       5.97s

  `fastmath=True`                     Yes      1.8s

  `parallel=True`                     No       1.36s

  `parallel=True`                     Yes      0.624s

  `parallel=True, fastmath=True`      No       1.32s

  `parallel=True, fastmath=True`      Yes      0.576s
  ----------------------------------------------------------------

SVML이 상당히 성능 향상을 이끌어 낸다는 것은 명백하다. 
SVML이 안 쓰이는 경우에 `fastmath`을 사용한 효과는 거의 없었다.
이점은 원래 함수에서 수치 제약만 해제하는 것이 별로 이득이 없음을 알려준다.

선형 대수
==============

Numba `numpy.linalg` 패키지의 대부분 함수는 no python mode를 지원한다.
내부 구현은 LAPACK과 BLAS 라이브러리를 이용하는 SciPy 함수 바인딩을 활용한다.
그런 까닭에 잘 최적화된 LAPACK/BLAS로 빌드된 SciPy를 사용하는 것이 필수적이다.
Anaconda 배포본의 경우 SciPy는 인텔의 MKL 기반으로 빌드되어 있는데,
그 결과 Numba는 고성능의 선형 대수를 수행할 수 있다.

[numba-parallel-supported]: /dev/numba_user_parallel#numba-parallel-supported "지원되는 연산"

**Note:** 
이 글은 Numba user manual을 번역한 글입니다.
[목차](/dev/numba_user_index)를 보려면 [여기](/dev/numba_user_index)를 클릭하세요.
{: .notice--warning}
