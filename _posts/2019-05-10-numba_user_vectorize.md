---
title: "[Numba 사용자 매뉴얼] 1.6 Numpy 유니버설 함수 만들기" 
strapline: "Numba의 vectorize는 스칼라 입력 인수를 취하는 파이썬 함수를 Numpy ufuncs으로 사용될 수 있도록 한다."
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

## `@vectorize` 데코레이터 {#the-vectorize-decorator}

Numba의 vectorize는 스칼라 입력 인수를 취하는 파이썬 함수를 Numpy 
[ufuncs](http://docs.scipy.org/doc/numpy/reference/ufuncs.html)으로 사용될 수 있도록 한다.
Numpy ufunc을 만드는 것은 보통 C 코드 작성과 연관이 되어 있어서 쉬운 과정이 아니다.
Numba는 이 작업을 쉽게 만든다.
[numba.vectorize](https://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#numba.vectorize) 데코레이터를 사용하여 
Numba는 순수 파이썬 함수를 ufunc 으로 컴파일할 수 있다.
이렇게 해서 만들어진 ufunc은 C로 쓰여진 전통적인 ufunc 만큼 빠르게 Numpy 배열을 다룬다.

[numba.vectorize](https://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#numba.vectorize) 데코레이터를 사용하여 
배열보다는 스칼라를 입력으로 취하는 함수를 만들자.
Numba는 실제 입력에 대해 효율적인 반복 루프(*커널*)를 생성할 것이다.

[numba.vectorize](https://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#numba.vectorize) 데코레이터는 두 가지 모드를 가지고 있다:

-   데코레이션-타임 컴파일레이션: 데코레이터에게 하나 이상의 타입 시그너쳐를 넘기면, Numpy 유니버설 함수(ufunc)를 작성하고 있는 것입니다. 이번절의 나머지는 데코레이션-타임 컴파일레이션을 사용한 ufunc 만들기를 기술한다.
-   연기된, 호출-타임 컴파일레이션: 어떤 시그너쳐도 넘기지 않으면, 데코레이터는 당신에게 Numba 동적 유니버설 함수
    [numba.DUFunc](https://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#numba.DUFunc)를 줄것이다.
    Numba 동적 유니버설 함수는 새롭게 출현한 입력 타입과 함께 호출되면 그때서야 동적으로 새로운 커널을 컴파일한다.
    \"[dynamic-universal-functions](numba_user_vectorize.html#dynamic-universal-functions)\" 절에서 좀더 깊게 이 모드를 기술한다.

위에서 언급된 것처럼 
[numba.vectorize](https://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#numba.vectorize) 데코레이터에 시그너쳐를 넘기면, 
당신의 함수는 Numpy ufunc으로 컴파일될 것이다. 
단지 한 개의 시그너쳐만 넘겨지는 기초적인 경우를 보자:

```python
    from numba import vectorize, float64

    @vectorize([float64(float64, float64)])
    def f(x, y):
        return x + y
```

당신이 여러개의 시그너쳐를 넘긴다면 가장 자세한 시그너쳐를 덜 자세한 시그너쳐보다 앞서서 넘겨야 한다
(예를 들어 단정도 실수가 배정도 실수보다 앞에 있어야 한다).
그렇지 않다면 타입 처리가 기대대로 작동하지 않을 것이다:

```python
    @vectorize([int32(int32, int32),
                int64(int64, int64),
                float32(float32, float32),
                float64(float64, float64)])
    def f(x, y):
        return x + y
```

함수는 지정된 배열 타입에 대해 기대한 대로 동작한다:

```python
    >>> a = np.arange(6)
    >>> f(a, a)
    array([ 0,  2,  4,  6,  8, 10])
    >>> a = np.linspace(0, 1, 6)
    >>> f(a, a)
    array([ 0. ,  0.4,  0.8,  1.2,  1.6,  2. ])
```

그러나, 다른 타입에 대해서는 에러가 난다:

```python
    >>> a = np.linspace(0, 1+1j, 6)
    >>> f(a, a)
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    TypeError: ufunc 'ufunc' not supported for the input types, and the inputs could not be safely coerced to any supported types according to the casting rule ''safe''
```

당신은 스스로에게 "왜 내가 `@jit` 데코레이터를 사용하면서 단순 반복 루틴을 컴파일하지 않고 이런 복잡한 것을 해야하는가?"라고 물어볼지도 모른다. 
그에 대한 대답은 NumPy ufunc은 축소(reduction), 누적(accumulation) 또는 브로드캐스팅과 같은 다른 특징들을
자동적으로 지원한다는 것이다.
위의 예를 사용하여: 

```python
    >>> a = np.arange(12).reshape(3, 4)
    >>> a
    array([[ 0,  1,  2,  3],
           [ 4,  5,  6,  7],
           [ 8,  9, 10, 11]])
    >>> f.reduce(a, axis=0)
    array([12, 15, 18, 21])
    >>> f.reduce(a, axis=1)
    array([ 6, 22, 38])
    >>> f.accumulate(a)
    array([[ 0,  1,  2,  3],
           [ 4,  6,  8, 10],
           [12, 15, 18, 21]])
    >>> f.accumulate(a, axis=1)
    array([[ 0,  1,  3,  6],
           [ 4,  9, 15, 22],
           [ 8, 17, 27, 38]])
```

**Callout:** [ufunc의 표준 특징들](http://docs.scipy.org/doc/numpy/reference/ufuncs.html#ufunc)(NumPy 문서)을 참고한다.
{: .notice--info}

[numba.vectorize](https://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#numba.vectorize) 데코레이터는 여러 ufunc 타겟을 지원한다:

| 타겟                | 설명           | 
|----|----|
| cpu                   | Single-threaded CPU   |
|----|----|
| parallel              | Multi-core CPU        |  
|----|----|
| cuda                  | CUDA GPU. 이 것은 유사 ufunc (ufunc-like) 개체를 생성한다. 자세한 것은 [CUDA ufunc 문서](https://numba.pydata.org/numba-doc/latest/cuda/ufunc.html)를 참조한다. |

일반적인 가이드라인은 데이터 크기와 알고리즘에 따라서 다른 타겟을 선택하는 것이다.
"cpu" 타겟은 작은 데이터(대략 1KB 이하) 및 저강도 계산 알고리즘에 대해서 잘 작동한다.
또한 오버헤드도 가장 작다. 
"parallel" 타겟은 중간 크기의 데이터(대략 1MB)에 대해서 잘 작동한다.
쓰레딩은 딜레이를 조금 발생시킨다.
"cuda" 타겟은 큰 데이터(대략 1MB이상) 및 고강도 계산 알고리즘에 대해서 잘 작동한다.
메모리를 GPU에 복사해가고 가져오는 등의 상당한 오버헤드가 추가된다.

## `@guvectorize` 데코레이터 {#guvectorize}

[numba.vectorize()](https://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#numba.vectorize)가 한 번에 한 요소씩 작업하는 ufunc을 작성하는데 쓰이는 반면에
[numba.guvectorize()](http://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#numba.guvectorize) 데코레이터는 한 걸음 더 나아가서 입력 배열의 임의의 수의 요소들에 대해서 작동하는 ufunc을 작성하는 데 쓰인다.
또한 다른 차원의 배열을 취하거나 리턴할 수 도 있다.
전형적인 예는 메디안 또는 컨볼루션 필터를 실행하는 것이다.

[numba.vectorize](https://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#numba.vectorize) 함수와는 다르게 [numba.guvectorize](http://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#numba.guvectorize) 함수는 결과값을 반환하지 않는다:
대신에 배열 인수를 취하고, 함수는 해당 배열에 결과를 저장한다.
이것은 해당 배열이 실질적으로 NumPy 디스패치 원리에 의해 할당되기 때문이다.

여기에 매우 간단한 예제가 있다.

```python
    @guvectorize([(int64[:], int64, int64[:])], '(n),()->(n)')
    def g(x, y, res):
        for i in range(x.shape[0]):
            res[i] = x[i] + y
```

위의 파이썬 함수는 단순히 일차원 배열의 모든 요소에 주어진 스칼라(`y`)를 더한다.
좀더 흥미로운 것은 함수 선언에 있다.
거기에는 두 개가 있다:

-   심볼 형태의 입력과 출력 *레이아웃* 선언: `(n),()->(n)`은 NumPy에게 함수는 *n*-개 요소 일차원 배열와 스칼라(빈 튜플 `()`로 표기된)를 취해서
    *n*-개 요소의 일차원 배열을 반환한다; 
-   `@vectorize`와 유사하게, 지원되는 구체적인 *시그니처* 목록; 여기서는 유일하게 `int64` 배열만 지원된다.

**Note:** 1D 배열 타입은 또한 스칼라 인수(`()` 모양을 가진 것들)를 받아 들일수 있다. 위의 예제에서 두번째 인수는 또한 `int64[:]`로 선언될 수 있었다. 그 경우, 값은 `y[0]`로 액세스된다.
{: .notice--warning}

간단한 예제를 통해, 이제 컴파일된 ufunc이 하는 것을 체크해보자: 

```python
    >>> a = np.arange(5)
    >>> a
    array([0, 1, 2, 3, 4])
    >>> g(a, 2)
    array([2, 3, 4, 5, 6])
```

나이스한 일은 Numpy가 자동으로 훨씬 복잡한 입력도 shape을 고려하여 디스패치한다는 것이다:

```python
    >>> a = np.arange(6).reshape(2, 3)
    >>> a
    array([[0, 1, 2],
           [3, 4, 5]])
    >>> g(a, 10)
    array([[10, 11, 12],
           [13, 14, 15]])
    >>> g(a, np.array([10, 20]))
    array([[10, 11, 12],
           [23, 24, 25]])
```

**Note:** numba.vectorize와 numba.guvectorize 모두 `@jit` 데코레이터처럼 "nopython=True"를 지원한다. 해당 옵션을 켜서 생성된 코드가 자동으로 개체 모드로 fallback하지 않도록 할 수 있다.
{: .notice--warning}


## 동적 유니버설 함수

앞서 언급한 것처럼 [numba.vectorize](http://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#numba.vectorize) 데코레이터에 어떠한 시그너처도 넘기지 않는다면,
당신의 파이썬 함수는 [numba.DUFunc](http://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#numba.DUFunc)라는 동적 유니버설 함수로 빌드될 것이다.
예를 들면: 

```python
    from numba import vectorize

    @vectorize
    def f(x, y):
        return x * y
```

결과적으로 `f()`는 지원되는 입력 타입 없이 시작하는 [numba.DUFunc](http://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#numba.DUFunc) 인스턴스가 된다.
당신이 `f()`를 호출할 때 비로소 Numba는 이전까지 지원하지 않았던 입력 타입을 넘길 때마다 새로운 커널을 생성한다.
위의 예의 경우, 아래와 같은 인터프리터 동작 내용은 동적 컴파일이 어떻게 작동하지 보여준다:

```python
    >>> f
    <numba._DUFunc 'f'>
    >>> f.ufunc
    <ufunc 'f'>
    >>> f.ufunc.types
    []
```

위의 예는 [numba.DUFunc](http://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#numba.DUFunc) 인스턴스가 ufunc이 아니라는 것을 보여준다.
ufunc의 서브클래스이기 보다는 [numba.DUFunc](http://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#numba.DUFunc) 인스턴스는 [ufunc](http://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#numba.DUFunc.ufunc) 멤버에 ufunc을 저장하고
ufunc 관련 속성과 호출을 이 멤버에 전달함으로써 (타입 집적 기술) ufunc처럼 동작한다.
ufunc에 의해 지원되는 입력 타입을 조사하면 처음에는 아무 것도 없음을 알 수 있다.

`f()`를 호출해보자:

```python
    >>> f(3,4)
    12
    >>> f.types   # shorthand for f.ufunc.types
    ['ll->l']
```

이 함수가 보통의 Numpy ufunc이었다면 해당 함수가 입력 타입을 다룰 수 없다는 불평을 하면서 예외를 던졌을 것이다.
그러나 이 경우에는 정수 인수로 `f()`를 부를 때 우리는 답을 받을 수 있을 뿐만 아니라
Numba가 `long` 정수를 지원하는 루프를 생성했음을 알 수 있다.

우리는 다른 입력 타입으로 `f()`를 호출함으로써 또 다른 루프를 생성할 수 있다:

```python
    >>> f(1.,2.)
    2.0
    >>> f.types
    ['ll->l', 'dd->d']
```

우리는 이제 Numba가 실수 입력 `"dd->d"`을 다룰 수 있는 두번째 루프를 추가했음을 알 수 있다.

`f()` 호출시에 입력 타입을 혼합한다고 하더라도 [Numpy ufunc 캐스팅 규칙](http://docs.scipy.org/doc/numpy/reference/ufuncs.html#casting-rules)에 따라 여전히 잘 적용된다:

```python
    >>> f(1,2.)
    2.0
    >>> f.types
    ['ll->l', 'dd->d']
```

이 예는 `f()`를 타입을 혼합해서 호출하면 Numpy가 실수 루프를 선택하여 정수 인자를 실수값으로 변환함을 보여준다.
그러므로 Numba는 특별히 `"dl->d"` 커널을 생성하거나 하지 않았다.

이런 [numba.DUFunc](http://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#numba.DUFunc)의 작동 방식은 [@vectorize 데코레이터](#the-vectorize-decorator) 절에서 주어진 것과 비슷한
경고가 여기에도 적용된다.
데코레이터에서는 시그너처 선언 순서가 중요하다면, 여기서는 호출 순서가 중요하다.
우리가 실수 인수를 먼저 전달했다면, 정수 인수를 가진 어떠한 호출도 실수 값으로 캐스팅되어 처리되었을 것이다.
예를 들어:

```python
    >>> @vectorize
    ... def g(a, b): return a / b
    ...
    >>> g(2.,3.)
    0.66666666666666663
    >>> g(2,3)
    0.66666666666666663
    >>> g.types
    ['dd->d']
```

다양한 타입 시그너처에 대해서 각각에 대해 정확한 커널이 필요하다면
[numba.vectorize](http://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#numba.vectorize) 데코레이터처럼 그들을 미리 지정하고, 동적 컴파일에 의존해서는 안 된다.
