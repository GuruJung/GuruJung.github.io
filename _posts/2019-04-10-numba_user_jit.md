---
title: '@jit과 함께 파이썬 코드 컴파일하기'
strapline: "Numba는 코드 생성을 위한 몇가지 유틸리티를 제공하는데, 그중에서도 중심은 numba.jit() 데코레이터이다."
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

Numba는 코드 생성을 위한 몇가지 유틸리티를 제공하는데, 그중에서도 중심은 [numba.jit()](http://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#numba.jit) 데코레이터이다.
본 데코레이터를 사용하여 Numba의 JIT 컴파일러로 최적화할 수 있는 함수를 마크할 수 있다.
다양한 호출 모드로 서로 다른 컴파일러 옵션과 행동을 지시할 수 있다.

## 기본 사용법

### 지연 컴파일 {#jit-lazy}
----------------

`@jit` 데코레이터를 사용할 때 추천하는 방법은 Numba로 하여금 최적화 시기와 방법을 스스로 결정토록 하는 것이다:

```python
    from numba import jit

    @jit
    def f(x, y):
        # A somewhat trivial example
        return x + y
```

이 모드로 컴파일은 첫번째 함수 실행때까지 연기된다. 
호출 시점에 Numba는 인수 타입을 추론하고 추론된 정보를 기반으로 최적화된 코드를 생성한다.
Numba는 또한 입력 타입에 따라 별도의 전용 코드를 컴파일할 수 있다.
예를 들면, 정수 또는 복소수를 가지고 위의 `f()` 함수를 호출할 때 서로 다른 코드를 생성할 것이다:

    >>> f(1, 2)
    3
    >>> f(1j, 2)
    (2+1j)

### 적극적 컴파일

또한 당신은 Numba에게 실행할 함수의 시그너처를 미리 알려줄 수도 있다. 
함수 `f()`는 아래와 같은 모습도 가능하다:

```python
    from numba import jit, int32

    @jit(int32(int32, int32))
    def f(x, y):
        # A somewhat trivial example
        return x + y
```

`int32(int32, int32)`이 바로 함수의 시그너처이다. 
이 경우에는 `@jit` 데코레이터가 (타입 추론 없이) 해당 시그너처에 대응되는 코드를 생성하여 컴파일한다.
이것은 컴파일러에 의해 선택되는 타입에 대해 사용자가 (예를 들어, 단정도 실수를 사용하도록 하는) 미세 조정을 하고자 할 때 유용하다.

`int32(int32, int32)` 대신에 `(int32, int32)`를 쓰는 식으로 리턴 타입을 생략하면, Numba는 리턴 타입에 대해서는 추론을 하려 한다.
함수 시그너처는 복수의 문자열도 가능한데, 리스트 형태로 그들을 건네줄 수 있다;
자세한 것은 [numba.jit](http://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#numba.jit) 문서를 참조한다.

물론 컴파일된 함수는 예상한 결과를 준다:

    >>> f(1,2)
    3

만약 리턴 타입으로서 `int32`를 지정하였다면, 고차원 비트는 버려진다:

    >>> f(2**31, 2**31 + 1)
    1

## 다른 함수를 호출하거나 인라이닝하기

Numba로 컴파일된 함수는 컴파일된 다른 함수를 호출할 수 있다.
해당 함수 호출은 최적화 휴리스틱에 의해 네이티브 코드안으로 아예 인라이닝될 수 있다.
예를 들어:

```python
    @jit
    def square(x):
        return x ** 2

    @jit
    def hypot(x, y):
        return math.sqrt(square(x) + square(y))
```

square와 같은 라이브러리 함수들에는 모두 `@jit` 데코레이터가 *붙어야* 한다.
그렇지 않으면 Numba는 hypot에 대해 훨씬 더 느린 코드를 생성하게 된다.

## 시그너처 사양

명시적인 `@jit` 시그너처는 많은 타입을 사용할 있는데 자주 사용하는 것들은 아래와 같다:

-   `void`는 아무 것도 리턴하지 않는 함수의 리턴 타입이다 (실제로는 파이썬으로부터 호출될 때 `None`을 리턴한다)
-   `intp`와 `uintp`는 각각 signed, unsiged 타입의, 포인터와 같은 크기의 정수 타입이다 (역자주: 정수 포인터를 표현할 때 사용한다)
-   `intc`와 `uintc`는 C의 `int`와 `unsigned int` 정수 타입에 해당한다
-   `int8`, `uint8`, `int16`, `uint16`, `int32`, `uint32`, `int64`, `uint64` 모두 해당 비트 너비를 가진 고정폭 정수 타입이다.
-   `float32`와 `float64`는 단정도 및 배정도 실수 타입이다
-   `complex64`는 `complex128`는 단정도 및 배정도 복소수 타입이다
-   배열 타입은 어떠한 숫자 타입을 인덱싱함으로써 지정될 수 있다. 
    예를들어, 일차원 단정도 실수 배열에 대해서는 `float32[:]`가, 이차원 8비트 정수 배열에 대해서는 `int8[:,:]`처럼 지정된다

## 컴파일 옵션 

다양한 키워드-온리 인수가 `@jit` 데코레이터에 건네질 수 있다.

### `nopython` {#jit-nopython}

Numba는 두 가지 컴파일 모드를 가지고 있다: [nopython 모드](http://numba.pydata.org/numba-doc/latest/glossary.html#term-nopython-mode)와 [object 모드](http://numba.pydata.org/numba-doc/latest/glossary.html#term-object-mode).
전자는 훨씬 더 빠른 코드를 생성하지만, 안 좋은 코딩 생성 조건에서는 후자로 빠진다.
Numba가 후자로 빠지게 하지 않고, 대신에 에러를 올리게 하려면, `nopython=True`을 건넨다.

```python
    @jit(nopython=True)
    def f(x, y):
        return x + y
```

**Note:** [문제 해결 및 팁](/dev/numba_user_troubleshoot#numba-troubleshooting)
{: .notice--warning}

### `nogil` {#jit-nogil}

Numba가 파이썬 코드를 네이티브 코드로 변환하고 최적화할 때
파이썬 개체가 아닌 네이티브 타입과 변수에서만 동작하는 데에 성공하면, 이제 더 이상 파이썬의 
[global interpreter lock](https://docs.python.org/3/glossary.html#term-global-interpreter-lock)을 걸 필요가 없다.
 `nogil=True`가 건네진다면 컴파일된 함수로 진입할 때 Numba는 GIL을 풀어줄 것이다.

```python
    @jit(nogil=True)
    def f(x, y):
        return x + y
```

GIL이 풀린 상태로 실행되는 코드는 파이썬이나 Numba 코드(동일한 함수이든 다른 것이든)를 실행하는 다른 쓰레드와 병행하여 실행된다.
이것은 멀티-코어 시스템에서 많은 이점이 있다.
그런데 함수가 `object 모드`로 빠져 컴파일된다면 GIL이 풀리지 않는다.

`nogil=True`를 사용할 때는 멀티-쓰레드 프로그래밍시 일어나는 위험들(일관성, 동기화, 경쟁 조건 등)에 대해서 조심해야 한다.

### `cache` {#jit-cache}

파이썬 프로그램을 실행할 때마다 컴파일이 일어나는 것을 피하기 위해서
Numba에게 함수 컴파일 결과를 파일 기반의 캐쉬로 쓰도록 지시할 수 있다.
`cache=True`를 건네주면 된다:

```python
    @jit(cache=True)
    def f(x, y):
        return x + y
```

### `parallel` {#parallel_jit_option}

병렬화하기 쉬운 논리를 가진 함수에서 해당 로직을 자동 병렬화(와 관련된 최적화)를 하도록 활성화한다.
지원되는 연산 목록을 볼려면 [병렬화](/dev/numba_user_parallel#numba-parallel)를 참조한다.
본 특징은 `parallel=True`를 건넴으로써 활성화되고 `nopython=True`와 같이 쓰여야 한다:

```python
    @jit(nopython=True, parallel=True)
    def f(x, y):
        return x + y
```

**Note:** [@jit과 함께하는 자동 병렬화](/dev/numba_user_parallel#numba-parallel)
{: .notice--warning}
