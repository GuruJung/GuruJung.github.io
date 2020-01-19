---
title: '[Numba 사용자 매뉴얼] 1.10 `@jit`과 함께 하는 자동 병렬화'
strapline: "numba.jit을 위한 병렬 옵션은 함수(일부분)를 자동적으로 병렬화 및 최적화를 할려고 시도하는 Numba 변환 패스를 활성화시킨다."
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

[jit][numba.jit]을 위한 [parallel][parallel_jit_option] 옵션을 켜면 Numba 변환 패스를 활성화시켜서 함수 일부분을 자동적으로 병렬화 및 최적화를 한다.
현재 이 기능은 CPU 타겟일 때만 작동한다.

배열에 스칼라 값을 더하는 연산은 병렬 의미(parallel semantics)를 가지고 있다고 알려져 있다.
사용자 프로그램은 일반적으로 그런 연산을 매우 많이 포함하고 있다.
그런 연산 각각을 개별로 병렬화시키는 접근은 캐시 문제로 인해 낮은 성능을 보인다.
대신에 자동 병렬화를 활성화할 때 Numba는 사용자 프로그램에서 그런 연산들을 발견하고 인접한 것들을 묶으려는 시도를 하여
한 개이상의 커널이 자동으로 병렬로 실행되도록 한다.
이런 과정은 사용자 프로그램의 수정없이 완전 자동으로 수행된다.
이런 특징은 병렬 커널을 만드는데 사람의 노력이 필요한 Numba의 [vectorize][numba.vectorize]나 [guvectorize][numba.guvectorize]의 메커니즘과 대비된다.

## 지원되는 연산 {#numba-parallel-supported}

본 절에서 우리가 현재 지원하는 병렬 의미를 가지고 있는 모든 배열 연산의 목록을 나열한다.

1.  [케이스 스터디: 배열 표현][case-study-array-expressions]에 의해 지원되는 모든 numba 배열 연산.
    Numpy 배열간, 배열과 스칼라간의 공통 산수 함수와 Numpy ufunc도 포함한다.
    이들은 자주 요소별 또는 포인트별 배열 연산이라고 불리기도 한다:

    > -   단항 연산자: `+` `-` `~`
    > -   이항 연산자: `+` `-` `*` `/` `/?` `%` `|` `>>` `^` `<<` `&` `**` `//`
    > -   비교 연산자: `==` `!=` `<` `<=` `>` `>=`
    > -   [nopython 모드]에서 지원되는 [Numpy ufunc][Numpy ufuncs]
    > -   [numba.vectorize]를 통한 사용자 정의 [numba.DUFunc]

2.  Numpy 집계 함수 `sum`, `prod`, `min`, `max`, `argmin`, `argmax`. 또한, 배열 수학 함수 `mean`, `var`, `std`.
3.  Numpy 배열 생성 함수 `zeros`, `ones`, `arange`,`linspace` 
    및 일부 랜덤 함수들(rand, randn, ranf, random_sample, sample, random, standard_normal, chisquare,
    weibull, power, geometric, exponential, poisson, rayleigh, normal,
    uniform, beta, binomial, f, gamma, lognormal, laplace, randint,
    triangular).
4.  행렬과 벡터간 또는 두 벡터간 Numpy `dot` 함수. 그 이외의 경우, Numba의 기본 구현이 사용된다.
5.  다차원 배열 또한 위의 연산들에 대해서 지원되는데, 인수의 차원과 크기는 매치가 되어야 한다. 
    차원이나 크기가 섞인 배열이 있을 때 일어나는 Numpy 브로드캐스트의 모든 기능이 다 지원되지도 않고, 선택된 차원에 대한 집계(reduction)도 지원되지 않는다. 
6.  타겟이 슬라이스 또는 부울 배열을 사용하는 배열 선택이고, 할당되는 값이 스칼라이거나 또다른 선택(슬라이스 범위 또는 비트 배열이 호환 가능한)인 배열 할당.
7.  `functools`의 `reduce` 연산자 또한 지원되는데, 1D Numpy 배열이어야 하고 초기 값은 필수이다.


## 명시적인 병렬 루프 {#numba-prange}

본 코드 변환 패스의, 또하나의 특징은 명시적인 병렬 루프를 지원한다는 것이다.
루프가 병렬화되도록 하기 위해서는 `range` 대신에 `prange`를 사용하면 된다.
지원되는 집계 함수를 제외하고는 사용자는 필수적으로 cross iteration 의존성이 없다는 것을 확인해야 한다.

루프 몸체에서 현재 변수가 이전 값을 사용하여 이진 함수/연산을 통해 갱신된다면, 집계(reduction)는 자동으로 추론된다.
`+=`과 `*=`에 대해서는 집계시의 초기 값이 자동으로 추론되지만
다른 함수/연산에 대해서는 집계 변수가 `prange` 루프에 들어가기 전에 수동으로 값이 초기화되어 있어야 한다.
이런 방법으로 스칼라나 임의 차원의 배열에 대해서 집계 연산이 지원된다.

아래 예제는 집계를 하는 병렬 루프를 보여준다(`A`는 일차원 Numpy 배열이다):

```python
    from numba import njit, prange
    @njit(parallel=True)
    def prange_test(A):
        s = 0
        for i in prange(A.shape[0]):
            s += A[i]
        return s
```

다음 예제는 이차원 배열에 대한 곱셈 집계를 보여준다:

```python
    from numba import njit, prange
    import numpy as np

    @njit(parallel=True)
    def two_d_array_reduction_prod(n):
        shp = (13, 17)
        result1 = 2 * np.ones(shp, np.int_)
        tmp = 2 * np.ones_like(result1)

        for i in prange(n):
            result1 *= tmp

        return result1
```

## 예제

본 절에서는 이런 기능이 어떻게 로지스틱 회귀를 도와주는 지의 예제를 보여주고 있다:

```python
    @numba.jit(nopython=True, parallel=True)
    def logistic_regression(Y, X, w, iterations):
        for i in range(iterations):
            w -= np.dot(((1.0 / (1.0 + np.exp(-Y * np.dot(X, w))) - 1.0) * Y), X)
        return w
```

회귀 알고리즘을 상세히 논의하지는 않고, 대신에 이 프로그램이 자동-병렬화와 어떻게 작동하는지에 대해서 초점을 맞춘다:

1.  입력 `Y` 는 크기 `N`인 벡터이고, `X`는 `N x D` 행렬이며, `w`는 크기 `D`인 벡터이다.
2.  함수 몸체는 변수 `w`를 갱신하는 반복 루프이다. 
    루프 몸체는 일련의 벡터와 행렬 연산으로 구성되어 있다.
3.  내적 `dot` 연산은 크기 `N`의 벡터를 만들어 내고, 그 후 스칼라와 크기 `N`인 벡터간 또는 크기 `N`인 두 벡터간, 일련의 수치 연산을 한다.
4.  외적 `dot`은 크기 `D`의 벡터를 생산해내고, 변수 `w`를 빼는 제자리(inplace) 배열 연산을 한다.
5.  자동-병렬화로, 크기 `N`의 배열을 만들어내는 모든 연산은 단일 병렬 커널이 되도록 함께 결합된다.
    이는 내적 `dot` 연산과 그 후의 모든 포인트별 배열 연산을 가리킨다.
6.  외적 `dot` 연산은 다른 차원의 배열을 반환하기 때문에 위의 커널과는 결합되지 않는다.

여기서, 병렬 하드웨어의 이점을 살리기 위해 요구되는 것은 단 하나, [jit][numba.jit]을 위한 [parallel][parallel_jit_option] 옵션을 설정하는 것이다.
`logistic_regression` 함수 그 자체는 수정할 필요가 없다.
만약에 [guvectorize][numba.guvectorize]를 사용하여 동일하게 병렬 구현을 하려면 
병렬화될 수 있는 커널을 추출하기 위해서 코드를 광범위하게 고쳐야 하고, 이는 매우 지루하면서도 어려운 일이다.


## 진단

**Note:** 
현재는 모든 병렬 변환 및 함수가 코드 생성 처리 동안에 추적될 수 있는 게 아니다.
때때로 어떤 루프나 변환에 대한 진단을 빠뜨린다.
{: .notice--warning}

[jit][numba.jit]에 [parallel][parallel_jit_option] 옵션의 적용은 데코레이트되는 코드를 자동적으로 병렬화하는 변환에 대한 진단 정보를 만들어 낼 수 있다.
이 정보는 두 가지 방법으로 접근될 수 있는데, 하나는 환경 변수 [NUMBA_PARALLEL_DIAGNOSTICS]를 설정하는 것이고
다른 하나는 [Dispatcher.parallel_diagnostics]를 호출하는 것이다.
두 방법 모두 같은 정보를 제공하며 `STDOUT`에 정보 출력한다.
진단 정보에서 출력량의 수준은 1과 4사이의 정수값으로 제어가 된다. 
1은 가장 출력량이 적고, 4는 가장 많다.
예를 들어:

```python
    @njit(parallel=True)
    def test(x):
        n = x.shape[0]
        a = np.sin(x)
        b = np.cos(a * a)
        acc = 0
        for i in prange(n - 2):
            for j in prange(n - 1):
                acc += b[i] + b[j + 1]
        return acc

    test(np.arange(10))

    test.parallel_diagnostics(level=4)
```

위 코드를 아래 문구를 출력한다.

```
    ================================================================================
    ======= Parallel Accelerator Optimizing:  Function test, example.py (4)  =======
    ================================================================================


    Parallel loop listing for  Function test, example.py (4) 
    --------------------------------------|loop #ID
    @njit(parallel=True)                  | 
    def test(x):                          | 
        n = x.shape[0]                    | 
        a = np.sin(x)---------------------| #0
        b = np.cos(a * a)-----------------| #1
        acc = 0                           | 
        for i in prange(n - 2):-----------| #3
            for j in prange(n - 1):-------| #2
                acc += b[i] + b[j + 1]    | 
        return acc                        | 
    --------------------------------- Fusing loops ---------------------------------
    Attempting fusion of parallel loops (combines loops with similar properties)...
    Trying to fuse loops #0 and #1:
        - fusion succeeded: parallel for-loop #1 is fused into for-loop #0.
    Trying to fuse loops #0 and #3:
        - fusion failed: loop dimension mismatched in axis 0. slice(0, x_size0.1, 1)
    != slice(0, $40.4, 1)
    ----------------------------- Before Optimization ------------------------------
    Parallel region 0:
    +--0 (parallel)
    +--1 (parallel)


    Parallel region 1:
    +--3 (parallel)
    +--2 (parallel)


    --------------------------------------------------------------------------------
    ------------------------------ After Optimization ------------------------------
    Parallel region 0:
    +--0 (parallel, fused with loop(s): 1)


    Parallel region 1:
    +--3 (parallel)
    +--2 (serial)



    Parallel region 0 (loop #0) had 1 loop(s) fused.

    Parallel region 1 (loop #3) had 0 loop(s) fused and 1 loop(s) serialized as part
    of the larger parallel loop (#3).
    --------------------------------------------------------------------------------
    --------------------------------------------------------------------------------

    ---------------------------Loop invariant code motion---------------------------

    Instruction hoisting:
    loop #0:
    Failed to hoist the following:
        dependency: $arg_out_var.10 = getitem(value=x, index=$parfor__index_5.99)
        dependency: $0.6.11 = getattr(value=$0.5, attr=sin)
        dependency: $expr_out_var.9 = call $0.6.11($arg_out_var.10, func=$0.6.11, args=[Var($arg_out_var.10, example.py (7))], kws=(), vararg=None)
        dependency: $arg_out_var.17 = $expr_out_var.9 * $expr_out_var.9
        dependency: $0.10.20 = getattr(value=$0.9, attr=cos)
        dependency: $expr_out_var.16 = call $0.10.20($arg_out_var.17, func=$0.10.20, args=[Var($arg_out_var.17, example.py (8))], kws=(), vararg=None)
    loop #3:
    Has the following hoisted:
        $const58.3 = const(int, 1)
        $58.4 = _n_23 - $const58.3
    --------------------------------------------------------------------------------
```

[parallel_jit_option] 옵션이 사용될 때 취해지는 변환에 익숙치 않는 사용자를 도와주고
계속되는 다음 절을 잘 이해하도록 도와주기 위해서 아래와 같은 정의를 제공한다:

 - 루프 융합(fusion)
  : [루프 융합](https://en.wikipedia.org/wiki/Loop_fission_and_fusion)은
    동일한 바운드를 가진 루프들을, 특정 조건을 만족해서, 더 큰 몸체를 가진 루프로 합치는 기술이다
    (데이터 국지성을 향상시키려는 목적으로).

 - 루프 직렬화
  : 루프 직렬화는 한 개이상의 `prange` 루프가 또다른 `prange` 루프안에 존재할 때 발생한다.
    이 경우에 가장 바깥쪽의 모든 `prange` 루프는 병렬로 실행되고, 안쪽의 `prange` 루프 (중첩되어 있든 아니든)는 표준 `range` 기반 루프로 취급돤다.
    즉, 중첩(nested) 병렬화는 일어 나지 않는다.

 - 루프 불변 코드 이동
  : [Loop invariant code motion](https://en.wikipedia.org/wiki/Loop-invariant_code_motion)은
    루프를 분석하여 루프의 실행 결과를 변경하지 않고도 루프 바깥으로 옮길 수 있는 코드 문장을 찾는 최적화 기법이다.
    이러한 문장은 반복 계산을 하지 않기 위해서 루프 바깥으로 "끌어 올려지게" 된다.
    
- 할당 끌어올리기
  : 할당 끌어올리기(allocation hoisting)는 루프 불변 코드 이동의 특별한 경우로써
    공통적으로 쓰이는 Numpy 할당 메쏘드의 설계에 기인하여 가능하다.
    이 기술의 설명은 예제로 가장 잘 설명된다.:

    ```python
    @njit(parallel=True)
    def test(n):
        for i in prange(n):
            temp = np.zeros((50, 50)) # <--- Allocate a temporary array with np.zeros()
            for j in range(50):
                temp[j, j] = i

        # ...do something with temp
    ```

    내부적으로, 이 코드는 대략 다음처럼 변형된다:

    ```python
    @njit(parallel=True)
    def test(n):
        for i in prange(n):
            temp = np.empty((50, 50)) # <--- np.zeros() is rewritten as np.empty()
            temp[:] = 0               # <--- and then a zero initialisation
            for j in range(50):
                temp[j, j] = i

        # ...do something with temp
    ```

    끌어올림 이후:

    ```python
    @njit(parallel=True)
    def test(n):
        temp = np.empty((50, 50)) # <--- allocation is hoisted as a loop invariant as `np.empty` is considered pure
        for i in prange(n):
            temp[:] = 0           # <--- this remains as assignment is a side effect
            for j in range(50):
                temp[j, j] = i

        # ...do something with temp
    ```

    `np.zeros` 할당이 할당과 치환으로 분리되는 것으로 보여질 수 있고, 할당은 `i` 루프 바깥으로끌어 올려졌다.
    이는 할당이 단 한번만 이루이지기 때문에 좀더 효율적인 코드를 생산한다.


### 병렬 진단 보고

보고서는 아래와 같이 여러 절로 분리된다:

1. 코드 주석(Code annotation)
    :   병렬 의미(parallel semantics)가 확인되고 나열될 루프를 가진 함수의 소스코드를 포함하는, 첫번째 절이다.
        소스코드 오른쪽에 있는 `loop #ID` 컬럼은 확인된 병렬 루프를 보여준다.
        예를 들어, `#0`는 `np.sin`이고, `#1`은 `np.cos`이며, `#2`와 `#3`은 `prange()`이다:

        ```
        Parallel loop listing for  Function test, example.py (4) 
        --------------------------------------|loop #ID
        @njit(parallel=True)                  | 
        def test(x):                          | 
            n = x.shape[0]                    | 
            a = np.sin(x)---------------------| #0
            b = np.cos(a * a)-----------------| #1
            acc = 0                           | 
            for i in prange(n - 2):-----------| #3
                for j in prange(n - 1):-------| #2
                    acc += b[i] + b[j + 1]    | 
            return acc                        | 
        ```

        여기서 언급할 것은 루프 아이디가 소스에서 나타나는 순서가 아닌, 병렬 의미를 가지고 있는 것이라고 발견되는 순서로 매겨진다는 것이다.
        더 나아가서, 병렬 변환은 스태틱 카운터를 사용하여 루프 아이디를 인덱싱한다는 것이다.
        그 결과 루프 아이디 인덱스는 0에서 출발하지 않는 경우도 있다. 
        사용자에게는 안 보이는 곳에서 일어나는 내부 최적화/변환 때도 같은 카운터를 사용하기 때문이다.

2. 루프 결합(Fusing loops)
    :   본 절은 발견된 병렬 루프들을 결합하려고 할 때 어느 루프는 성공하고 어느 루프는 실패하는 지를 기술한다.
        결합이 실패한 경우에는 그 이유가 주어진다 (e.g. 다른 데이터와의 의존성 때문).
        예를 들자면:

        ```python
        --------------------------------- Fusing loops ---------------------------------
        Attempting fusion of parallel loops (combines loops with similar properties)...
        Trying to fuse loops #0 and #1:
            - fusion succeeded: parallel for-loop #1 is fused into for-loop #0.
        Trying to fuse loops #0 and #3:
            - fusion failed: loop dimension mismatched in axis 0. slice(0, x_size0.1, 1)
        != slice(0, $40.4, 1)
        ```

        위의 경우, `#0`과 `#1` 루프의 결합이 시도되었고 성공하였다 (둘다 `x`와 같은 차원에 기반으로 한 것이다).
        이 성공이후 `#0` (`#1`을 포함한 있는)과 `#3` 사이에 결합을 시도하였다.
        이 결합은 실패하였는데, `#0`의 크기는 `x.shape`인 반면에 `#3`의 크기는 `x.shape[0] - 2`이어서
        루프 차원에 미스매치가 있었기 때문이다.

3.  최적화 전
    :   본 절은 최적화가 이루어지기 전의 코드상 병렬 영역의 구조를 보여준다
        (최적화 전후의 출력값을 직접 비교하기 위해서 보여주는 것이다).
        결합될 수 없는 루프들이 있다면 복수의 병렬 영역이 존재할 수 있는데,
        이 경우에 각 영역내의 코드가 병렬로 실행되지만 각 병렬 영역은 순차적으로 실행된다.
        위의 예로부터:

        ```
        Parallel region 0:
        +--0 (parallel)
        +--1 (parallel)


        Parallel region 1:
        +--3 (parallel)
        +--2 (parallel)
        ```

        루프 결합 절에서 암시되었듯이, 필수적으로 코드안에 두 개의 병렬 영역이 있다.
        첫번째 영역은 루프 `#0`과 `#1`을 포함하고, 두번째 영역은 `#3`과 `#2`를 포함한다.
        아직 어떤 최적화도 일어 나지 않았기 때문에 모든 루프는 `parallel`이라고 마크된다. 

4. 최적화 후 
    :   본 절은 최적화가 이루어진 후의 병렬 영역의 구조를 보여준다.
        다시 말하지만, 병렬 영역은 해당 영역과 매치되는 루프와 함께 나열되는데,
        지금은 결합되거나 순차적인 루프를 마크되고, 요약을 보여준다.
        위의 예로부터: 

        ```
        Parallel region 0:
        +--0 (parallel, fused with loop(s): 1)


        Parallel region 1:
        +--3 (parallel)
           +--2 (serial)
        ```

        병렬 영역 0 (루프 #0)은 한개의 루프를 결합하였다.

        병렬 영역 1 (루프 #3)은 0개의 루프를 결합하였고, (상대적으로 더 큰 병렬 로프 #3의 일부로서) 1개의 순차 루프를 가지고 있다. 

        병렬 영역 0은 루프 `#0`을 포함하고, 루프 결합 섹션에서 보여지듯이 루프 `#1`은 루프 `#0`안으로 융합된다.
        병렬 영역 1은 루프 `#3`을 포함하고 있고, 루프 `#2`(내부 `prange()`)는 루프 `#3`의 몸체에서 순차적으로 실행된다.

5. 루프 불변 코드 이동 
    :   본 절은 최적화가 이루어진 후의 각 루프에 대해서 아래 항목을 보여준다:

        -   끌어 올려지는 데 실패한 명령과 실패한 이유 (의존성/불순물)
        -   끌어 올려지는 데 성공한 명령
        -   일어날 수도 있었던, 할당 끌어올림

        위의 예로부터:

        ```
        Instruction hoisting:
        loop #0:
        Failed to hoist the following:
            dependency: $arg_out_var.10 = getitem(value=x, index=$parfor__index_5.99)
            dependency: $0.6.11 = getattr(value=$0.5, attr=sin)
            dependency: $expr_out_var.9 = call $0.6.11($arg_out_var.10, func=$0.6.11, args=[Var($arg_out_var.10, example.py (7))], kws=(), vararg=None)
            dependency: $arg_out_var.17 = $expr_out_var.9 * $expr_out_var.9
            dependency: $0.10.20 = getattr(value=$0.9, attr=cos)
            dependency: $expr_out_var.16 = call $0.10.20($arg_out_var.17, func=$0.10.20, args=[Var($arg_out_var.17, example.py (8))], kws=(), vararg=None)
        loop #3:
        Has the following hoisted:
            $const58.3 = const(int, 1)
            $58.4 = _n_23 - $const58.3
        ```

        첫번째 유의 사항은 이 정보가 고급 사용자용이라는 것이다.
        왜냐하면 변환되고 있는 함수의 [Numba IR]을 참조하고 있기 때문이다.
        예로서, 예제 소스에서 `a * a` 표현은 IR에서 `$arg_out_var.17 = $expr_out_var.9 * $expr_out_var.9` 표현으로 부분적으로 번역된다. 
        이것은 명백히 루프 `#0`에서 끌어 올려질 수 없다.
        루프 불변이 아니기 때문이다.
        반면에 루프 `#3`처럼, 
        `$const58.3 = const(int, 1)` 표현은 소스 `b[j + 1]` 출신이다.
        숫자 `#1`은 명백히 상수이고 그래서 루프 바깥에서 끌어 올려질 수 있다.

**See also:** 
[parallel][parallel_jit_option], [병렬화 관련 FAQs][Parallel FAQs]
{: .notice--info}

[parallel_jit_option]: /dev/numba_user_jit#parallel-jit-option "parallel"
[numba.jit]: http://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#numba.jit "jit"
[numba.vectorize]: http://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#numba.vectorize
[numba.guvectorize]: http://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#numba.guvectorize
[case-study-array-expressions]: http://numba.pydata.org/numba-doc/latest/developer/rewrites.html#case-study-array-expressions 
[Numpy ufuncs]: http://numba.pydata.org/numba-doc/latest/reference/numpysupported.html#supported-ufuncs
[nopython 모드]: http://numba.pydata.org/numba-doc/latest/glossary.html#term-nopython-mode
[numba.DUFunc]: http://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#numba.DUFunc
[NUMBA_PARALLEL_DIAGNOSTICS]: http://numba.pydata.org/numba-doc/latest/reference/envvars.html#envvar-NUMBA_PARALLEL_DIAGNOSTICS
[Dispatcher.parallel_diagnostics]: http://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#Dispatcher.parallel_diagnostics
[Parallel FAQs]: /dev/numba_user_faq#parallel-faqs 
[Numba IR]: http://numba.pydata.org/numba-doc/latest/glossary.html#term-numba-ir


**Note:** 
이 글은 Numba user manual을 번역한 글입니다.
[목차](/dev/numba_user_index)를 보려면 [여기](/dev/numba_user_index)를 클릭하세요.
{: .notice--warning}
