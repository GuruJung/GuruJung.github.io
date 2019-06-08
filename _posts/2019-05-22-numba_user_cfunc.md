---
title: '[Numba 사용자 매뉴얼] 1.8 `@cfunc`으로 C 콜백 함수 만들기'
strapline: "파이썬으로 쓰여진 비즈니스 로직을 C나 C++로 쓰여진 네이티브 라이브러리에 제공하기 위해 콜백을 작성해야 하는 경우가 있다."
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

파이썬으로 쓰여진 비즈니스 로직을 C나 C++로 쓰여진 네이티브 라이브러리에 제공하기 위해 콜백을 작성해야 하는 경우가 있다.
이 때문에 네이티브 라이브러리와의 연동은 필수적이다.
`numba.cfunc()` 데코레이터는 사용자가 지정한 시그너처로 외부 C 코드로부터 호출가능한, 컴파일된 함수를 생성한다.

## 기본 사용법

`@cfunc` 데코레이터의 사용법은 `@jit`의 경우와 유사하지만 중요한 차이가 있다:
함수 시그너처를 반드시 전달해야 한다는 것이다.
해당 시그너처는 C 콜백 함수의 시그너처에 일대일로 대응된다:

```python
    from numba import cfunc

    @cfunc("float64(float64, float64)")
    def add(x, y):
        return x + y
```

C 함수 개체는 CFunc 개체으로서 [address](http://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#CFunc.address) 속성은 컴파일된 C 콜백 함수의 주소를 가리킨다.
그래서 그것을 외부 C/C++ 라이브러리에 전달할 수 있다.
또한 C 함수 개체는 [ctypes](https://docs.python.org/3/library/ctypes.html#module-ctypes) 콜백 개체도 노출시키는데, 콜백 개체는 또한 파이썬으로부터 호출가능하다.
이는 컴파일된 코드를 검사할 때 편리하다:

```python
    @cfunc("float64(float64, float64)")
    def add(x, y):
        return x + y

    print(add.ctypes(4.0, 5.0))  # prints "9.0"
```

## 예제

이 예제에서는 `scipy.integrate.quad` 함수를 사용하려 한다.
이 함수는 일반 파이썬 콜백 또는 [ctypes](https://docs.python.org/3/library/ctypes.html#module-ctypes) 콜백 개체로 래핑된 C 콜백을 취한다.

순서 파이썬 integrand를 정의하고 그것을 C콜백으로 컴파일하자:

```python
    >>> import numpy as np
    >>> from numba import cfunc
    >>> def integrand(t):
            return np.exp(-t) / t**2
       ...:
    >>> nb_integrand = cfunc("float64(float64)")(integrand)
```

`nb_integrand` 개체의 [ctypes](https://docs.python.org/3/library/ctypes.html#module-ctypes) 콜백을 `scipy.integrate.quad`에 전달하고,
그 결과가 순수 파이썬 함수를 전달한 것과 같은 결과를 보이는지 확인하자:

```python
    >>> import scipy.integrate as si
    >>> def do_integrate(func):
            """
            Integrate the given function from 1.0 to +inf.
            """
            return si.quad(func, 1, np.inf)
       ...:
    >>> do_integrate(integrand)
    (0.14849550677592208, 3.8736750296130505e-10)
    >>> do_integrate(nb_integrand.ctypes)
    (0.14849550677592208, 3.8736750296130505e-10)
```

컴파일된 콜백을 사용함으로써, integration 함수는 integrand를 평가할 때마다 파이썬 인터프리터를 호출할 필요가 없어졌다.
우리 경우에 integration은 18배 더 빨리 실행되었다:

```python
    >>> %timeit do_integrate(integrand)
    1000 loops, best of 3: 242 µs per loop
    >>> %timeit do_integrate(nb_integrand.ctypes)
    100000 loops, best of 3: 13.5 µs per loop
```

## 포인터와 배열 메모리 다루기

좀 더 복잡한 C콜백 이용 사례는 호출자로부터 전달된 데이터 배열에 대한 연산을 수행할 때이다.
C는 Numpy 배열과 같은 고수준의 추상화가 없기 때문에 C 콜백 시그너처는 저수준의 포인터와 배열 크기 인수를 전달할 것이다. 
그럼에도 불구하고, 콜백에 대한 파이썬 코드는 Numpy 배열과 같은 파워와 표현력을 기대한다.

다음 예제에서 C 콜백은 `void(double *input, double *output, int m, int n)` 시그너처로 2D 배열상에 작동하도록 기대된다.
아래와 같은 콜백을 구현할 수 있다:

```python
    from numba import cfunc, types, carray

    c_sig = types.void(types.CPointer(types.double),
                       types.CPointer(types.double),
                       types.intc, types.intc)

    @cfunc(c_sig)
    def my_callback(in_, out, m, n):
        in_array = carray(in_, (m, n))
        out_array = carray(out, (m, n))
        for i in range(m):
            for j in range(n):
                out_array[i, j] = 2 * in_array[i, j]
```

[numba.carray](http://numba.pydata.org/numba-doc/latest/reference/utils.html#numba.carray) 함수는 데이터 포인터와 모양(크기)를 취하여 해당 데이터에 대한, 주어진 모양의 배열 뷰를 리턴한다.
데이터는 C에서 사용하는 순서로 놓여져 있다고 가정한다. 
데이터가 포트란에서 사용하는 순서로 놓여져 있다면 [numba.farray](http://numba.pydata.org/numba-doc/latest/reference/utils.html#numba.farray)를 사용하여야 한다.

## 시그너처 사양

`@cfunc`에서 사용하는 시그너처는 아무 [Numba 타입](http://numba.pydata.org/numba-doc/latest/reference/types.html#numba-types)이나 이용할 수 있는데, 
실제로는 그 중 일부만 의미있게 쓰인다.
보통은 `int8`이나 `float64`와 같은 스칼라 타입이나 `types.CPointer(types.int8)`와 같은 그들에 대한 포인터 정도를 시그니처에 사용한다.

## 컴파일 옵션

많은 키워드 인수가 `@cfunc` 데코레이터에 전달될 수 있다: `nopython` and `cache`. 
`@jit` 데코레이터에서 쓰일 때와 의미가 비슷하다.


**Note:** 
이 글은 Numba user manual을 번역한 글입니다.
[목차](/dev/numba_user_index)를 보려면 [여기](/dev/numba_user_index)를 클릭하세요.
{: .notice--warning}
