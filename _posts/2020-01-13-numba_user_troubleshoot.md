---
title: '[Numba 사용자 매뉴얼] 1.15 트러블슈팅 및 팁'
strapline: "소스 코드 중에서 가장 성능이 중요한 부분만을 컴파일하려고 노력해야 한다."
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

어떤 것을 컴파일할까?
===============

소스 코드 중에서 가장 성능이 중요한 부분만을 컴파일하려고 노력해야 한다.
고수준의 코드내에서 성능이 중요한 부분을 가지고 있다면 그 부분을 별도의 함수로 떼어내고
Numba로 컴파일하는 것이 좋다.
Numba가 작은 크기의 코드에 집중하도록 함으로써 몇몇의 이점을 갖게 된다:

-   지원하지 못 하는 파트를 만나게 될 위험성을 줄인다.
-   컴파일 시간을 줄인다.
-   고수준의 코드를 좀더 잘 구조화시키는 데 도움을 준다.


코드 컴파일이 안 됩니다 {#code-doesnt-compile}
========================

Numba가 당신의 코드를 컴파일하지 못 하고 대신에 에러를 띄우는 이유는 다양하다.
그 중 자주 실수하는 파트는 Numba가 지원하지 못 하는 파이썬 특징을 사용했기 때문이다.
특히 [nopython-mode]에서 지원되지 못 하는 특징을 사용했기 때문이다.
[pysupported] 목록을 보기 바랍니다.
거기에서 언급된 것인데 여전히 컴파일에 실패한다면 [report-numba-bugs]를 해주십시요.

Numba가 코드를 컴파일할 때 우선 사용중인 모든 변수의 타입을 구할려고 한다.
그렇게 함으로써 타입이 구체화된 구현된 코드를 생성할 수 있고 곧 기계어 코드로 컴파일할 수 있다.
Numba가 컴파일에 실패하는 주요 이유(특히 [nopython-mode]에서 실패하는 이유)는 타입을 추론하는데 실패했기 때문이다.
특히, 필수적으로 알아내야 하는 변수의 타입을 알아낼 수 없을 때 그렇다.

예를 들어 다음과 같은 작고 단순한 함수를 보자:

```python
    @jit(nopython=True)
    def f(x, y):
        return x + y
```

두개의 숫자와 함께 그것을 호출할 때 Numba는 그 타입을 적절히 추론할 수 있다:

```python
    >>> f(1, 2)
        3
```

그러나 튜플과 숫자하나로 호출한다면 Numba는 튜플과 숫자를 더한 결과가 무엇인지 말할 수 없을 것이고
결국 컴파일 에러를 낸다:

```
    >>> f(1, (2,))
    Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
    File "<path>/numba/numba/dispatcher.py", line 339, in _compile_for_args
        reraise(type(e), e, None)
    File "<path>/numba/numba/six.py", line 658, in reraise
        raise value.with_traceback(tb)
    numba.errors.TypingError: Failed at nopython (nopython frontend)
    Invalid use of + with parameters (int64, tuple(int64 x 1))
    Known signatures:
    * (int64, int64) -> int64
    * (int64, uint64) -> int64
    * (uint64, int64) -> int64
    * (uint64, uint64) -> uint64
    * (float32, float32) -> float32
    * (float64, float64) -> float64
    * (complex64, complex64) -> complex64
    * (complex128, complex128) -> complex128
    * (uint16,) -> uint64
    * (uint8,) -> uint64
    * (uint64,) -> uint64
    * (uint32,) -> uint64
    * (int16,) -> int64
    * (int64,) -> int64
    * (int8,) -> int64
    * (int32,) -> int64
    * (float32,) -> float32
    * (float64,) -> float64
    * (complex64,) -> complex64
    * (complex128,) -> complex128
    * parameterized
    [1] During: typing of intrinsic-call at <stdin> (3)

    File "<stdin>", line 3:
```

에러 메시지는 무엇이 잘못 되었는지 알려준다: 
\"Invalid use of + with parameters (int64, tuple(int64 x 1))\"는 
Numba가 정수와 정수의 1-tuple의 합을 마주치게 되었는데 그런 연산에 대해서는 모른다는 뜻이다.

만약 object mode를 허용한다면:

```python
    @jit
    def g(x, y):
        return x + y
```

컴파일은 성공할 것이나 컴파일된 함수는 파이썬 해석기가 그렇게 하듯 실행시간에 예외를 발생시킨다: 

```
    >>> g(1, (2,))
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    TypeError: unsupported operand type(s) for +: 'int' and 'tuple'
```

타입 통일 문제
======================================

Numba가 코드 컴파일을 하지 못하는 또다른 주요 원인은 함수의 리턴 타입을 정적으로 결정할 수 없는 경우에 발생한다.
오직 실행 시간때에만 알수 있는 값에 의존하는 리턴 타입일 경우에 많이 일어난다.
다시 말해서, [nopython-mode]에서 가장 자주 일어나는 문제이다.
타입 통일이라는 개념은 간단히 말해서 두 개의 변수가 안전하게 표현될 수 있는 타입을 구할려고 하는 것이다.
예를 들어 64비트 실수와 64비트 허수는 둘다 128비트 복소수로 표현될 수 있다.

타입 통일 실패의 예제로서, 이 함수는 \`x\` 값에 기반하여 실행시간에 결정되는 리턴 타입을 가지고 있다:

```
    In [1]: from numba import jit

    In [2]: @jit(nopython=True)
    ...: def f(x):
    ...:     if x > 10:
    ...:         return (1,)
    ...:     else:
    ...:         return 1
    ...:     

    In [3]: f(10)
```

이 함수를 실행하면 아래와 같은 에러가 발생한다:

```
    TypingError: Failed at nopython (nopython frontend)
    Can't unify return type from the following types: tuple(int64 x 1), int64
    Return of: IR name '$8.2', type '(int64 x 1)', location: 
    File "<ipython-input-2-51ef1cc64bea>", line 4:
    def f(x):
        <source elided>
        if x > 10:
            return (1,)
            ^
    Return of: IR name '$12.2', type 'int64', location: 
    File "<ipython-input-2-51ef1cc64bea>", line 6:
    def f(x):
        <source elided>
        else:
            return 1
```

에러 메시지 \"Can\'t unify return type from the following types: tuple(int64 x 1), int64\"은
Numba가 튜플과 정수를 안전하게 동시에 표현할 수 있는 타입을 발견할 수 없었다는 것이다.


타입이 정의 안된 리스트 문제 {#code-has-untyped-list}
===================================

코드 컴파일 실패의 주요 원인은 이전에 언급되었듯이 모든 변수의 타입이 무엇인지 알아내는 과정과 연관되어 있다.
리스트의 경우 같은 타입의 요소를 가지고 있어야 하거나 추후 진행되는 연산에서 타입이 추론될 수 있다면 빈 리스트도 허용된다.
타입을 추론할 수 없는 빈 리스트가 정의될 때 문제가 발생한다.

예를 들어 아래는 알려진 타입의 리스트를 사용한 경우이다:

```python
    from numba import jit
    @jit(nopython=True)
    def f():
        return [1, 2, 3] # this list is defined on construction with `int` type
```

본 예제는 빈 리스트이지만 타입이 추론될 수 있는 경우이다:

```python
    from numba import jit
    @jit(nopython=True)
    def f(x):
        tmp = [] # defined empty
        for i in range(x):
            tmp.append(i) # list type can be inferred from the type of `i`
        return tmp
```

본 예제는 빈 리스트이면서 타입은 추론되지 않는 경우이다:

```python
    from numba import jit
    @jit(nopython=True)
    def f(x):
        tmp = [] # defined empty
        return (tmp, x) # ERROR: the type of `tmp` is unknown
```

조금만 고민하면, 리스트가 비어 있고 타입이 추론될 수 없지만 당신은 어떤 타입인지 이미 알고 있다면,
아래 트릭을 사용하여 타입 추론 메커니즘에 힌트를 줄 수 있다:

```python
    from numba import jit
    import numpy as np
    @jit(nopython=True)
    def f(x):
        # define empty list, but instruct that the type is np.complex64
        tmp = [np.complex64(x) for x in range(0)]
        return (tmp, x) # the type of `tmp` is known, but it is still empty
```

컴파일된 코드가 너무 느리다
=============================

컴파일된 JIT 함수가 느린 주요 원인은 [nopython-mode]에서의 컴파일이 실패해서
자동으로 [object-mode]로 빠져서 일반 파이썬 해석기와 비교해서 속도 향상이 거의 없는 코드로 컴파일되었기 때문이다.
[object-mode]의 주요 포인트는 [loop-lifting]로 알려진 내부 최적화를 허용하는 것이다:
이런 최적화는 내부 루프를 둘러싼 코드가 무엇이든가에 [nopython-mode]로 내부 루프를 컴파일하도록 해준다.

타입 추론이 성공적이었는지 알아내려면 [dispatcher-inspect-types] 메쏘드를 사용하면 된다.

예를 들어 다음과 같은 함수를 보자:

```python
    @jit
    def f(a, b):
        s = a + float(b)
        return s
```

숫자와 함께 호출되었을 때 본 함수는 Numba가 두 숫자 타입을 실수로 변환할 수 있기 때문에 매우 빠르다.
아래를 보자:

```
    >>> f(1, 2)
    3.0
    >>> f.inspect_types()
    f (int64, int64)
    --------------------------------------------------------------------------------
    # --- LINE 7 ---

    @jit

    # --- LINE 8 ---

    def f(a, b):

        # --- LINE 9 ---
        # label 0
        #   a.1 = a  :: int64
        #   del a
        #   b.1 = b  :: int64
        #   del b
        #   $0.2 = global(float: <class 'float'>)  :: Function(<class 'float'>)
        #   $0.4 = call $0.2(b.1, )  :: (int64,) -> float64
        #   del b.1
        #   del $0.2
        #   $0.5 = a.1 + $0.4  :: float64
        #   del a.1
        #   del $0.4
        #   s = $0.5  :: float64
        #   del $0.5

        s = a + float(b)

        # --- LINE 10 ---
        #   $0.7 = cast(value=s)  :: float64
        #   del s
        #   return $0.7

        return s
```

Numba의 중간 내부 표현을 많이 모르더라도 모든 변수와 임시 값들이 그들의 타입이 적절히 추론됨을 알 수 있다:
예를 들어 *a*는 타입 `int64`, *0.5*는 타입 `float64`이다.

그러나 *b*가 문자열로서 전달된다면, 문자열을 가진 float()는 Numba가 지원하지 않기 때문에 object mode로 빠진다: 

```
    >>> f(1, "2")
    3.0
    >>> f.inspect_types()
    [... snip annotations for other signatures, see above ...]
    ================================================================================
    f (int64, str)
    --------------------------------------------------------------------------------
    # --- LINE 7 ---

    @jit

    # --- LINE 8 ---

    def f(a, b):

        # --- LINE 9 ---
        # label 0
        #   a.1 = a  :: pyobject
        #   del a
        #   b.1 = b  :: pyobject
        #   del b
        #   $0.2 = global(float: <class 'float'>)  :: pyobject
        #   $0.4 = call $0.2(b.1, )  :: pyobject
        #   del b.1
        #   del $0.2
        #   $0.5 = a.1 + $0.4  :: pyobject
        #   del a.1
        #   del $0.4
        #   s = $0.5  :: pyobject
        #   del $0.5

        s = a + float(b)

        # --- LINE 10 ---
        #   $0.7 = cast(value=s)  :: pyobject
        #   del s
        #   return $0.7

        return s
```

여기에서 우리는 모든 변수가 `pyobject`로 타입이 바뀐 것을 알 수 있다.
이것은 함수가 object 모드에서 컴파일되었음을 의미하고, Numba는 더 깊게 조사하려고 시도하지 않는다.
이것은 코드 실행 속도를 생각할 때 피하고 싶은 상황일 것이다.

nopython 모드에서 함수 컴파일이 실패하는 원인을 이해하기 위한 방법이 몇개 있다:

-   *nopython=True*를 전달하여 뭐가 잘못 되어 가고 있는지 바로 에러를 뜨도록 한다;
-   [NUMBA_WARNINGS] 환경 변수를 설정하여 경고를 활성화한다; 예를 들어 `f()`  함수의 경우:

```
>>> f(1, 2)
3.0
>>> f(1, "2")
example.py:7: NumbaWarning: Function "f" failed type inference: Internal error at <numba.typeinfer.CallConstrain object at 0x7f6b8dd24550>:
float() only support for numbers
File "example.py", line 9
  @jit
example.py:7: NumbaWarning: Function "f" was compiled in object mode without forceobj=True.
  @jit
3.0
```

JIT 컴파일 비활성화하기
=========================

코드를 디버그하기 위해서 JIT 컴파일을 비활성화하는 것이 가능하다.
즉, `jit` 데코레이터 (`njit` 및 `autojit` 모두)가 아무 일도 안 하는 것처럼 행동한다.
데코레이트되는 함수를 호출하면 그냥 원래의 파이썬 함수를 호출한다.
[NUMBA_DISABLE_JIT] 환경 변수를 `1`로 설정함으로써 이루어진다.

이 모드가 활성화되더라도 `vectorize`와 `guvectorize` 데코레이터는 여전히 ufunc을 컴파일할 것이다.
왜냐하면 이들 함수의 순수 파이썬 함수 버전은 존재하지 않기 때문이다.

GDB로 JIT 컴파일된 코드를 디버깅하기
====================================

`jit` 데코레이터에 `debug` 키워드를 세팅하면 (e.g. `@jit(debug=True)`),
컴파일된 코드에 디버그 정보를 표출하도록 설정된다.
디버그하기 위해서 GDB 7.0 이상의 버전이 요구된다.
현재 아래와 같은 디버그 정보가 가능하다:

-   추적시에 함수 이름이 보이지만, 타입 정보는 안 보인다.
-   소스 위치 (파일 이름과 줄번호)가 보인다. 
    예를 들어 사용자는 절대 경로의 파일 이름과 줄번호로 브레이크 포인트를 설정할 수 있다; 
    e.g. `break /path/to/myfile.py:6`.
-   `info locals`로 현재 함수의 지역 변수가 보여진다.
-   `whatis myvar`로 변수 타입을 보여준다.
-   `print myvar` 또는 `display myvar`를 가지고 변수의 값을 볼 수 있다.
    -   정수, 실수와 같은 단순한 수치 타입은 바로 보여진다. 다만, 정수는 signed 타입으로 가정되어 값이 보인다.
    -   다른 타입은 바이트 배열로서 보인다.

알려진 이슈:

-   진행 단계는 최적화 레벨에 따라 진행 정도가 크게 차이난다.
    -   완전한 최적화 (O3에 동급)의 경우에 대부분의 변수가 최적화되어 사라진다.
    -   최적화가 없을 때 (e.g. `NUMBA_OPT=0`) 소스 위치는 코드 주위로 점프한다.
    -   O1 최적화시 (e.g. `NUMBA_OPT=1`), 진행은 안정적이나 어떤 변수들은 최적화되어 사라진다.
-   디버그 정보가 활성화되었을 때 메모리 소비가 크게 늘어난다.
    컴파일러는 부가 정보([DWARF])를 발생시킨다.
    생성된 오브젝트 코드는 디버그 정보를 가지고 있어서 2배 더 커진다.

내부 세부 사항:

-   파이썬 의미론에서는 변수가 타입이 다른 값들에 바인딩되는 것을 허용하기 때문에,
    Numba는 내부적으로 하나의 변수에 대해서 개별 타입에 대응되는 복수개의 변수 이름을 생성한다.
    그래서 아래와 같은 코드의 경우:

```python
        x = 1         # type int
        x = 2.3       # type float
        x = (1, 2, 3) # type 3-tuple of int
```

    각 할당은 다른 변수 이름으로 저장될 것이다.
    디버거에서 변수들은 `x`, `x$1`, `x$2`가 될 것이고, Numba IR에서는 각각 `x`, `x.1`, `x.2`가 된다.

-   디버그가 활성화될 때 함수 인라인화는 비활성화된다.

디버그 사용 예제
-------------------

파이썬 소스:

```python
from numba import njit

@njit(debug=True)
def foo(a):
    b = a + 1
    c = a * 2.34
    d = (a, b, c)
    print(a, b, c, d)

r= foo(123)
print(r)
```

터미널:

```
$ NUMBA_OPT=1 gdb -q python
Reading symbols from python...done.
(gdb) break /home/user/chk_debug.py:5
No source file named /home/user/chk_debug.py.
Make breakpoint pending on future shared library load? (y or [n]) y

Breakpoint 1 (/home/user/chk_debug.py:5) pending.
(gdb) run chk_debug.py
Starting program: /home/user/miniconda/bin/python chk_debug.py
...
Breakpoint 1, __main__::foo$241(long long) () at chk_debug.py:5
5         b = a + 1
(gdb) n
6         c = a * 2.34
(gdb) bt
#0  __main__::foo$241(long long) () at chk_debug.py:6
#1  0x00007ffff7fec47c in cpython::__main__::foo$241(long long) ()
#2  0x00007fffeb7976e2 in call_cfunc (locals=0x0, kws=0x0, args=0x7fffeb486198,
...
(gdb) info locals
a = 0
d = <error reading variable d (DWARF-2 expression error: `DW_OP_stack_value' operations must be used either alone or in conjunction with DW_OP_piece or DW_OP_bit_piece.)>
c = 0
b = 124
(gdb) whatis b
type = i64
(gdb) whatis d
type = {i64, i64, double}
(gdb) print b
$2 = 124
```


디버그 설정을 전역으로 덮어 쓰기
-------------------------------

`NUMBA_DEBUGINFO=1` 환경 변수 설정으로 전체 앱에 대해서 디버그를 활성화시킬 수 있다.
이것은 `jit`에 기본적으로 `debug` 옵션이 추가되는 것처럼 해준다.
이 상태에서 개별 함수에 대해서 디버그 옵션을 끌려면 `debug=False`를 전달하면 된다.

메모리 소비를 많이 하기 때문에 이 옵션을 켜는 것에 주의를 해야 한다.
큰 앱의 경우 메모리 에러가 날수도 있다.

`nopython` 모드에서 직접 `gdb` 바인딩 사용하기 
=======================================================

Numba (0.42.0 이후)는 CPU 타겟에 대해서 `gdb`를 연동시키는 함수들을 제공하고 있는데,
이것은 프로그램 디버그를 좀더 쉽게 한다.
아래에서 언급되는 `gdb` 관련 함수들은 표준 파이선 해석기 내에서 또는 nopython/object 모드에서 컴파일된 코드 모두에서
똑같은 방식으로 작동한다.

**Note:** 
이 기능은 실험적이다.
{: .notice--warning}

**Warning:** 
이 기능은 Jupyter 또는 `pdb` 모듈에서 사용될 때 해롭지는 않지만 예견하기 힘든 행동을 벌인다.
{: .notice--warning}

준비
------

Numba의 `gdb` 관련 함수는 `gdb` 바이너리를 사용하기 때문에
`gdb` 실행 파일의 위치와 이름이 [NUMBA_GDB_BINARY] 환경 변수에 설정되어 있어야 한다.

**Note:** 
Numba의 `gdb` 지원은 `gdb`가 다른 프로세스에 attach할 수 있는 능력을 필요로 한다.
어떤 시스템(우분투 리눅스)은 기본적으로 `ptrace`에 보안 제약이 있어서 그런 능력을 막아 버린다.
이런 제약은 리눅스 보안 모듈 Yama에 의해 시스템 레벨에서 강제된다.
이 모듈에 대한 문서와 제약을 변경시키기 위한 방법은 [리눅스 커널 문서](https://www.kernel.org/doc/Documentation/admin-guide/LSM/Yama.rst)에
기록되어 있다.
[우분투 리눅스 보안 문서](https://wiki.ubuntu.com/Security/Features#ptrace)는 Yama의 행동을 변경시키는 방법을
이야기하고 있는데, 위 능력을 허용하기 위해서 필요한 `ptrace_scope` 관련 사항을 찾아 본다.
{: .notice--warning}


기본 `gdb` 지원
-------------------

**Warning:** 
동일한 프로그램내에서 `numba.gdb()` 또는 `numba.gdb_init()`을 한번이상 부르는 것은 좋지 않고, 예측할 수 없는 것들이 발생한다.
한 프로그램 내에서 여러개의 브레이크 포인트가 요구된다면,
`numba.gdb()` 또는 `numba.gdb_init()`을 통해 `gdb`를 한번 실행하고 
`numba.gdb_breakpoint()`를 통해 브레이크 포인트를 등록한다.
{: .notice--warning}

`gdb` 지원을 추가하기 위한 가장 간단한 함수는 `numba.gdb()`인데, 호출 위치에서:

-   `gdb`를 실행하여 현재 프로세스에 붙인다.
-   또한 중단점이 생성되어 부착된 `gdb`가 실행을 중지시키고 사용자 입력을 기다린다.

다음의 예를 보자:

```python
from numba import njit, gdb

@njit(debug=True)
def foo(a):
    b = a + 1
    gdb() # instruct Numba to attach gdb at this location and pause execution
    c = a * 2.34
    d = (a, b, c)
    print(a, b, c, d)

r= foo(123)
print(r)
```

터미널:

```
$ NUMBA_OPT=0 python demo_gdb.py
Attaching to PID: 27157
GNU gdb (GDB) Red Hat Enterprise Linux 8.0.1-36.el7
...
Attaching to process 27157
...
Reading symbols from <elided for brevity> ...done.
0x00007f0380c31550 in __nanosleep_nocancel () at ../sysdeps/unix/syscall-template.S:81
81      T_PSEUDO (SYSCALL_SYMBOL, SYSCALL_NAME, SYSCALL_NARGS)
Breakpoint 1 at 0x7f036ac388f0: file numba/_helperlib.c, line 1090.
Continuing.

Breakpoint 1, numba_gdb_breakpoint () at numba/_helperlib.c:1090
1090    }
(gdb) s
Single stepping until exit from function _ZN5numba7targets8gdb_hook8hook_gdb12$3clocals$3e8impl$242E5Tuple,
which has no line number information.
__main__::foo$241(long long) () at demo_gdb.py:7
7           c = a * 2.34
(gdb) l
2
3       @njit(debug=True)
4       def foo(a):
5           b = a + 1
6           gdb() # instruct Numba to attach gdb at this location and pause execution
7           c = a * 2.34
8           d = (a, b, c)
9           print(a, b, c, d)
10
11      r= foo(123)
(gdb) p a
$1 = 123
(gdb) p b
$2 = 124
(gdb) p c
$3 = 0
(gdb) n
8           d = (a, b, c)
(gdb) p c
$4 = 287.81999999999999
```

위 예에서 코드 실행은 `gdb()` 함수 호출 위치에서 멈춤을 알 수 있다.
`step`을 하면 컴파일된 소스 코드의 스택 프레임을 한 단계 진행한다.
그리고 `a`와 `b` 변수는 평가되어 값을 볼 수 있지만 `c`는 아직 그렇지 않다는 것을 알 수 있다
(역자주: 그래서 초기값 0을 보여준다).
이 것은 `gdb()` 호출 후에 예견되는 현상이다.
`next`를 하면 그때서야 라인 `7`을 평가하고 `c` 값이 계산된다.

`gdb`가 활성화된 채로 실행하기
--------------------------

`numba.gdb()`에 의해 제공되는 기능(`gdb`를 실행시키고 현재 프로세스에 부착시키고, 중단점상에 실행을 멈추게 하는 것)
은 또한 두 개의 별도의 함수로 구성된다:

-   `numba.gdb_init()` 함수는 호출 위치에 `gdb`를 실행하고 부착하지만 실행을 멈추게 하지는 않는다.
-   `numba.gdb_breakpoint()` 함수는 호출 위치에 특수한 `numba_gdb_breakpoint` 함수를 호출하는데,
    중단점을 등록한다. 이 것과 관련하여 다음 절에서 데모를 보여줄 것이다. 

이 기능은 좀더 복잡한 디버깅을 할 수 있다.
'segfault'(`SIGSEGV` 신호를 발생시키는 메모리 접근 에러)를 디버깅하는 예제를 보자:

```python
  from numba import njit, gdb_init
  import numpy as np

  @njit(debug=True)
  def foo(a, index):
      gdb_init() # instruct Numba to attach gdb at this location, but not to pause execution
      b = a + 1
      c = a * 2.34
      d = c[index] # access an address that is a) invalid b) out of the page
      print(a, b, c, d)

  bad_index = int(1e9) # this index is invalid
  z = np.arange(10)
  r = foo(z, bad_index)
  print(r)
```

터미널:

```
$ python demo_gdb_segfault.py
Attaching to PID: 5444
GNU gdb (GDB) Red Hat Enterprise Linux 8.0.1-36.el7
...
Attaching to process 5444
...
Reading symbols from <elided for brevity> ...done.
0x00007f8d8010a550 in __nanosleep_nocancel () at ../sysdeps/unix/syscall-template.S:81
81      T_PSEUDO (SYSCALL_SYMBOL, SYSCALL_NAME, SYSCALL_NARGS)
Breakpoint 1 at 0x7f8d6a1118f0: file numba/_helperlib.c, line 1090.
Continuing.

0x00007fa7b810a41f in __main__::foo$241(Array<long long, 1, C, mutable, aligned>, long long) () at demo_gdb_segfault.py:9
9           d = c[index] # access an address that is a) invalid b) out of the page
(gdb) p index
$1 = 1000000000
(gdb) p c
$2 = "p\202\017\364\371U\000\000\000\000\000\000\000\000\000\000\n\000\000\000\000\000\000\000\b\000\000\000\000\000\000\000\240\202\017\364\371U\000\000\n\000\000\000\000\000\000\000\b\000\000\000\000\000\000"
(gdb) whatis c
type = {i8*, i8*, i64, i64, double*, [1 x i64], [1 x i64]}
(gdb) x /32xb c
0x7ffd56195068: 0x70    0x82    0x0f    0xf4    0xf9    0x55    0x00    0x00
0x7ffd56195070: 0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x7ffd56195078: 0x0a    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x7ffd56195080: 0x08    0x00    0x00    0x00    0x00    0x00    0x00    0x00
```

`gdb` 출력에서 중단점으로서 `numba_gdb_breakpoint` 함수가 등록되어 있다는 것에 주목한다
(해당 함수는 `numba/_helperlib.c`에 있다).
`SIGSEGV` 시그널이 잡혔을 때 접근 에러가 일어났던 줄이 출력된다.

디버깅 세션 데모로써 예제를 계속하기 위해서 `index` 값을 출력하면 1,000,000,000 임을 알 수 있다.
`c` 값은 긴 길이의 바이트 값을 보여주기 때문에 좀더 조사가 필요하다.
`c`의 타입은 `DataModel` 기반으로 배열 `c`의 레이아웃을 보여준다.
`DataModel`은 레이아웃 관련하여 Numba 소스 `numba.datamodel.models`에 있다.
`ArrayModel` 또한 아래에 보여진다.

```python
class ArrayModel(StructModel):
    def __init__(self, dmm, fe_type):
        ndim = fe_type.ndim
        members = [
            ('meminfo', types.MemInfoPointer(fe_type.dtype)),
            ('parent', types.pyobject),
            ('nitems', types.intp),
            ('itemsize', types.intp),
            ('data', types.CPointer(fe_type.dtype)),
            ('shape', types.UniTuple(types.intp, ndim)),
            ('strides', types.UniTuple(types.intp, ndim)),
        ]
```

`gdb`로부터 조사된 타입 (`type = {i8*, i8*, i64, i64, double*, [1 x i64], [1 x i64]}`)은
`ArrayModel`의 멤버에 바로 대응된다.
잘못된 접근으로 segfault가 발생했을 때는 배열의 요소 개수를 체크하여 요구된 인덱스 값과 비교하는 것이 좋다. 

`c`의 메모리 (`x /32xb c`)를 조사하면 첫 16바이트는 두개의 `i8*`인데,
`meminfo` 포인터와 `parent` `pyobject`에 대응된다.
다음 8바이트짜리 두 개 그룹은 `i64`/`intp` 타입이고 `nitems`와 `itemsize`에 대응된다.
확실히 그들의 값은 `0x0a`와 `0x08`이다.
즉, 입력 배열 `a`는 10개의 요소이고 그것의 크기는 `int64`, 즉, 8바이트이다.
그런 까닭에 segfault는 10개 요소를 가진 배열에 엄청 큰 값의 index로 접근하려다 발생한 것임이 명백해진다.

코드에 중단점 추가하기
--------------------------

`numba.gdb_breakpoint()` 함수를 호출하여 여러 개의 중단점을 사용하는 예제를 보여준다:

```python
from numba import njit, gdb_init, gdb_breakpoint

@njit(debug=True)
def foo(a):
    gdb_init() # instruct Numba to attach gdb at this location
    b = a + 1
    gdb_breakpoint() # instruct gdb to break at this location
    c = a * 2.34
    d = (a, b, c)
    gdb_breakpoint() # and to break again at this location
    print(a, b, c, d)

r= foo(123)
print(r)
```

터미널:

```
$ NUMBA_OPT=0 python demo_gdb_breakpoints.py
Attaching to PID: 20366
GNU gdb (GDB) Red Hat Enterprise Linux 8.0.1-36.el7
...
Attaching to process 20366
Reading symbols from <elided for brevity> ...done.
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib64/libthread_db.so.1".
Reading symbols from /lib64/libc.so.6...Reading symbols from /usr/lib/debug/usr/lib64/libc-2.17.so.debug...done.
0x00007f631db5e550 in __nanosleep_nocancel () at ../sysdeps/unix/syscall-template.S:81
81      T_PSEUDO (SYSCALL_SYMBOL, SYSCALL_NAME, SYSCALL_NARGS)
Breakpoint 1 at 0x7f6307b658f0: file numba/_helperlib.c, line 1090.
Continuing.

Breakpoint 1, numba_gdb_breakpoint () at numba/_helperlib.c:1090
1090    }
(gdb) step
__main__::foo$241(long long) () at demo_gdb_breakpoints.py:8
8           c = a * 2.34
(gdb) l
3       @njit(debug=True)
4       def foo(a):
5           gdb_init() # instruct Numba to attach gdb at this location
6           b = a + 1
7           gdb_breakpoint() # instruct gdb to break at this location
8           c = a * 2.34
9           d = (a, b, c)
10          gdb_breakpoint() # and to break again at this location
11          print(a, b, c, d)
12
(gdb) p b
$1 = 124
(gdb) p c
$2 = 0
(gdb) continue
Continuing.

Breakpoint 1, numba_gdb_breakpoint () at numba/_helperlib.c:1090
1090    }
(gdb) step
11          print(a, b, c, d)
(gdb) p c
$3 = 287.81999999999999
```

`gdb` 출력으로부터 중단점이 히트되어 라인 8에서 실행이 멈추었고,
`continue` 후에 다시 다음 중단점에 히트되어 라인 11에서 다시 멈추었음을 볼 수 있다.

병렬 영역에서 하는 디버깅
-----------------------------

다음 예제는 위의 예제와 비슷한데, 쓰레드를 사용하고 중단점 기능을 사용한다.
더 나아가서, 병렬 영역의 마지막 반복에서 `work` 함수를 호출하는데,
그것은 실제로 `glibc`의 `free(3)`에 대한 바인딩이다.
그래서 segfault가 나게 된다.

```python
from numba import njit, prange, gdb_init, gdb_breakpoint
  import ctypes

  def get_free():
      lib = ctypes.cdll.LoadLibrary('libc.so.6')
      free_binding = lib.free
      free_binding.argtypes = [ctypes.c_void_p,]
      free_binding.restype = None
      return free_binding

  work = get_free()

  @njit(debug=True, parallel=True)
  def foo():
      gdb_init() # instruct Numba to attach gdb at this location, but not to pause execution
      counter = 0
      n = 9
      for i in prange(n):
          if i > 3 and i < 8: # iterations 4, 5, 6, 7 will break here
              gdb_breakpoint()

          if i == 8: # last iteration segfaults
              work(0xBADADD)

          counter += 1
      return counter

  r = foo()
  print(r)
```

`NUMBA_NUM_THREADS`을 4로 세팅하여 병렬 영역에서 실행되는 쓰레드가 4개임을 보장함을 유의하면서 터미널 출력을 본다:

```
$ NUMBA_NUM_THREADS=4 NUMBA_OPT=0 python demo_gdb_threads.py
Attaching to PID: 21462
...
Attaching to process 21462
[New LWP 21467]
[New LWP 21468]
[New LWP 21469]
[New LWP 21470]
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib64/libthread_db.so.1".
0x00007f59ec31756d in nanosleep () at ../sysdeps/unix/syscall-template.S:81
81      T_PSEUDO (SYSCALL_SYMBOL, SYSCALL_NAME, SYSCALL_NARGS)
Breakpoint 1 at 0x7f59d631e8f0: file numba/_helperlib.c, line 1090.
Continuing.
[Switching to Thread 0x7f59d1fd1700 (LWP 21470)]

Thread 5 "python" hit Breakpoint 1, numba_gdb_breakpoint () at numba/_helperlib.c:1090
1090    }
(gdb) info threads
Id   Target Id         Frame
1    Thread 0x7f59eca2f740 (LWP 21462) "python" pthread_cond_wait@@GLIBC_2.3.2 ()
    at ../nptl/sysdeps/unix/sysv/linux/x86_64/pthread_cond_wait.S:185
2    Thread 0x7f59d37d4700 (LWP 21467) "python" pthread_cond_wait@@GLIBC_2.3.2 ()
    at ../nptl/sysdeps/unix/sysv/linux/x86_64/pthread_cond_wait.S:185
3    Thread 0x7f59d2fd3700 (LWP 21468) "python" pthread_cond_wait@@GLIBC_2.3.2 ()
    at ../nptl/sysdeps/unix/sysv/linux/x86_64/pthread_cond_wait.S:185
4    Thread 0x7f59d27d2700 (LWP 21469) "python" numba_gdb_breakpoint () at numba/_helperlib.c:1090
* 5    Thread 0x7f59d1fd1700 (LWP 21470) "python" numba_gdb_breakpoint () at numba/_helperlib.c:1090
(gdb) thread apply 2-5 info locals

Thread 2 (Thread 0x7f59d37d4700 (LWP 21467)):
No locals.

Thread 3 (Thread 0x7f59d2fd3700 (LWP 21468)):
No locals.

Thread 4 (Thread 0x7f59d27d2700 (LWP 21469)):
No locals.

Thread 5 (Thread 0x7f59d1fd1700 (LWP 21470)):
sched$35 = '\000' <repeats 55 times>
counter__arr = '\000' <repeats 16 times>, "\001\000\000\000\000\000\000\000\b\000\000\000\000\000\000\000\370B]\"hU\000\000\001", '\000' <repeats 14 times>
counter = 0
(gdb) continue
Continuing.
[Switching to Thread 0x7f59d27d2700 (LWP 21469)]

Thread 4 "python" hit Breakpoint 1, numba_gdb_breakpoint () at numba/_helperlib.c:1090
1090    }
(gdb) continue
Continuing.
[Switching to Thread 0x7f59d1fd1700 (LWP 21470)]

Thread 5 "python" hit Breakpoint 1, numba_gdb_breakpoint () at numba/_helperlib.c:1090
1090    }
(gdb) continue
Continuing.
[Switching to Thread 0x7f59d27d2700 (LWP 21469)]

Thread 4 "python" hit Breakpoint 1, numba_gdb_breakpoint () at numba/_helperlib.c:1090
1090    }
(gdb) continue
Continuing.

Thread 5 "python" received signal SIGSEGV, Segmentation fault.
[Switching to Thread 0x7f59d1fd1700 (LWP 21470)]
__GI___libc_free (mem=0xbadadd) at malloc.c:2935
2935      if (chunk_is_mmapped(p))                       /* release mmapped memory. */
(gdb) bt
#0  __GI___libc_free (mem=0xbadadd) at malloc.c:2935
#1  0x00007f59d37ded84 in $3cdynamic$3e::__numba_parfor_gufunc__0x7ffff80a61ae3e31$244(Array<unsigned long long, 1, C, mutable, aligned>, Array<long long, 1, C, mutable, aligned>) () at <string>:24
#2  0x00007f59d17ce326 in __gufunc__._ZN13$3cdynamic$3e45__numba_parfor_gufunc__0x7ffff80a61ae3e31$244E5ArrayIyLi1E1C7mutable7alignedE5ArrayIxLi1E1C7mutable7alignedE ()
#3  0x00007f59d37d7320 in thread_worker ()
from <path>/numba/numba/npyufunc/workqueue.cpython-37m-x86_64-linux-gnu.so
#4  0x00007f59ec626e25 in start_thread (arg=0x7f59d1fd1700) at pthread_create.c:308
#5  0x00007f59ec350bad in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:113
```

출력을 보면, 4개의 쓰레드가 실행되었고, 중단점에서 모든 쓰레드가 중단되었다는 것을 알 수 있다.
더 나아가서 `Thread 5`는 `SIGSEGV` 신호를 받고 백트레이스를 보면 
`__GI___libc_free`에서 `mem`안의 주소가 잘못 되어서 신호가 발생했음을 알 수 있다.

`gdb` 명령 언어 사용하기
--------------------------------

`numba.gdb()`와 `numba.gdb_init()` 모두 무제한의 문자열 인수를 받아들여서 `gdb`에 직접 명령 라인 인수로서 전달한다.
이것은 중단점 설정이나 반복되는 디버깅 일을 매번 수동으로 할 필요없이 쉽게 할 수 있도록 도와준다.
예를 들면, 아래 코드는 `gdb`를 붙일 때 `_dgesdd`에 중단점을 설정한다
(예를 들어 LAPACK의 divide and conqueror SVD 함수에 전달되는 인수를 디버깅한다고 하자):

```python
  from numba import njit, gdb
  import numpy as np

  @njit(debug=True)
  def foo(a):
      # instruct Numba to attach gdb at this location and on launch, switch
      # breakpoint pending on , and then set a breakpoint on the function
      # _dgesdd, continue execution, and once the breakpoint is hit, backtrace
      gdb('-ex', 'set breakpoint pending on',
          '-ex', 'b dgesdd_',
          '-ex','c',
          '-ex','bt')
      b = a + 10
      u, s, vh = np.linalg.svd(b)
      return s # just return singular values

  z = np.arange(70.).reshape(10, 7)
  r = foo(z)
  print(r)
```

중단 및 백트레이스를 하는데 어떠한 인터랙션도 필요없음을 알수 있다:

```
$ NUMBA_OPT=0 python demo_gdb_args.py
Attaching to PID: 22300
GNU gdb (GDB) Red Hat Enterprise Linux 8.0.1-36.el7
...
Attaching to process 22300
Reading symbols from <py_env>/bin/python3.7...done.
0x00007f652305a550 in __nanosleep_nocancel () at ../sysdeps/unix/syscall-template.S:81
81      T_PSEUDO (SYSCALL_SYMBOL, SYSCALL_NAME, SYSCALL_NARGS)
Breakpoint 1 at 0x7f650d0618f0: file numba/_helperlib.c, line 1090.
Continuing.

Breakpoint 1, numba_gdb_breakpoint () at numba/_helperlib.c:1090
1090    }
Breakpoint 2 at 0x7f65102322e0 (2 locations)
Continuing.

Breakpoint 2, 0x00007f65182be5f0 in mkl_lapack.dgesdd_ ()
from <py_env>/lib/python3.7/site-packages/numpy/core/../../../../libmkl_rt.so
#0  0x00007f65182be5f0 in mkl_lapack.dgesdd_ ()
from <py_env>/lib/python3.7/site-packages/numpy/core/../../../../libmkl_rt.so
#1  0x00007f650d065b71 in numba_raw_rgesdd (kind=kind@entry=100 'd', jobz=<optimized out>, jobz@entry=65 'A', m=m@entry=10,
    n=n@entry=7, a=a@entry=0x561c6fbb20c0, lda=lda@entry=10, s=0x561c6facf3a0, u=0x561c6fb680e0, ldu=10, vt=0x561c6fd375c0,
    ldvt=7, work=0x7fff4c926c30, lwork=-1, iwork=0x7fff4c926c40, info=0x7fff4c926c20) at numba/_lapack.c:1277
#2  0x00007f650d06768f in numba_ez_rgesdd (ldvt=7, vt=0x561c6fd375c0, ldu=10, u=0x561c6fb680e0, s=0x561c6facf3a0, lda=10,
    a=0x561c6fbb20c0, n=7, m=10, jobz=65 'A', kind=<optimized out>) at numba/_lapack.c:1307
#3  numba_ez_gesdd (kind=<optimized out>, jobz=<optimized out>, m=10, n=7, a=0x561c6fbb20c0, lda=10, s=0x561c6facf3a0,
    u=0x561c6fb680e0, ldu=10, vt=0x561c6fd375c0, ldvt=7) at numba/_lapack.c:1477
#4  0x00007f650a3147a3 in numba::targets::linalg::svd_impl::$3clocals$3e::svd_impl$243(Array<double, 2, C, mutable, aligned>, omitted$28default$3d1$29) ()
#5  0x00007f650a1c0489 in __main__::foo$241(Array<double, 2, C, mutable, aligned>) () at demo_gdb_args.py:15
#6  0x00007f650a1c2110 in cpython::__main__::foo$241(Array<double, 2, C, mutable, aligned>) ()
#7  0x00007f650cd096a4 in call_cfunc ()
from <path>/numba/numba/_dispatcher.cpython-37m-x86_64-linux-gnu.so
...
```

`gdb` 바인딩은 어떻게 작동하는가?
--------------------------------

고급 사용자에게는 `gdb` 바인딩의 내부 구현을 조금 알 필요가 있다.
`numba.gdb()`와 `numba.gdb_init()` 함수는 아래와 같은 것들을 함수의 LLVM IR에 집어 넣음으로서 작동한다:

-   처음 함수 호출 위치에 `getpid(3)`에 대한 호출을 넣는다.
    이는 현재 프로세스의 PID를 저장하여 나중에 사용에 사용하기 위함이다.
    그후 `fork(3)` 호출을 집어 넣는다:
    -   부모 프로세스:
        -   `sleep(3)` 호출을 집어 넣는다 (그러므로 `gdb`가 로드되는 동안 잠깐 쉰다).
        -   `numba_gdb_breakpoint`에 대한 호출을 집어 넣는다 (`numba.gdb()`만이 이 일을 한다). 
    -   자식 프로세스:
        -   `numba.config.GDB_BINARY`, `attach` 명령어, 바로 전에 언급한 PID를 인수로 하는 `execl(3)`에 대한 호출을 집어 넣는다.
            Numba는 `numba_gdb_breakpoint` 심볼에 중단점을 설치하고 `finish`하는, 특별한 `gdb` 명령어 파일을 가지고 있다.
            이로 인해 프로그램은 중단점상에서 멈춘다.
            이 명령어 파일 또한 인수에 추가된다.

`numba.gdb_breakpoint()` 호출 위치에서, 중단하고 `finish`하는 위치로서 이미 등록되고 지시된, 특별한 `numba_gdb_breakpoint` 심볼에 
어떤 호출이 삽입이 된다.

그 결과 `numba.gdb()` 호출은 프로그램을 포킹하고, 부모는 잠들어 있는 동안에
자식은 `gdb`를 실행시켜 그것을 부모에 붙이고 나서 부모를 깨우는 일을 한다.
`gdb`는 중단점으로 설정된 `numba_gdb_breakpoint` 심볼을 가지고 있어서, 부모가 다시 시작할 때
즉시 `numba_gdb_breakpoint`를 호출한다.
`numba.gdb_breakpoint()` 호출은 결국 중단점에 의해 프로그램이 다시 정지되도록 할 것이다.


CUDA 파이썬 코드를 디버깅하기
==========================

시뮬레이터 사용하기
-------------------

CUDA 파이썬 코드는 CUDA 시뮬레이터를 통해 파이썬 해석기에서 실행될 수 있다.
이것은 파이썬 디버거 또는 print 문과 함께 디버그될 수 있음을 의미한다.
CUDA 시뮬레이터를 활성화하기 위해서는 환경 변수 [NUMBA_ENABLE_CUDASIM]을 1로 설정하면 된다.
CUDA 시뮬레이터에 대한 자세한 정보는 [CUDA Simulator documentation]을 참조하세요.

디버그 정보
----------

`cuda.jit`에 `debug` 인수를 `True`로 설정함으로써 (`@cuda.jit(debug=True)`),
Numba는 컴파일된 CUDA code에 소스 위치를 보여주는 코드가 들어가 있을 것이다.
CPU 타겟과는 달리 오직 파일 이름과 라인 정보만이 나오고 타입 정보는 안 나온다.
[cuda-memcheck](http://docs.nvidia.com/cuda/cuda-memcheck/index.html)를 이용하면
메모리 에러를 디버그하기에 충분하다.

예를 들어, 다음과 같은 cuda 파이썬 코드를 생각해 보자:

```python
import numpy as np
from numba import cuda

@cuda.jit(debug=True)
def foo(arr):
    arr[cuda.threadIdx.x] = 1

arr = np.arange(30)
foo[1, 32](arr)   # more threads than array elements
```

`cuda-memcheck`를 사용하여 메모리 에러를 찾자:

```
$ cuda-memcheck python chk_cuda_debug.py
========= CUDA-MEMCHECK
========= Invalid __global__ write of size 8
=========     at 0x00000148 in /home/user/chk_cuda_debug.py:6:cudapy::__main__::foo$241(Array<__int64, int=1, C, mutable, aligned>)
=========     by thread (31,0,0) in block (0,0,0)
=========     Address 0x500a600f8 is out of bounds
...
=========
========= Invalid __global__ write of size 8
=========     at 0x00000148 in /home/user/chk_cuda_debug.py:6:cudapy::__main__::foo$241(Array<__int64, int=1, C, mutable, aligned>)
=========     by thread (30,0,0) in block (0,0,0)
=========     Address 0x500a600f0 is out of bounds
...
```

[nopython-mode]: https://numba.pydata.org/numba-doc/latest/glossary.html#term-nopython-mode  "nopython 모드"
[pysupported]: https://numba.pydata.org/numba-doc/latest/reference/pysupported.html#pysupported "지원"
[report-numba-bugs]: https://numba.pydata.org/numba-doc/latest/developer/contributing.html#report-numba-bugs "버그 리포트"
[loop-lifting]: https://numba.pydata.org/numba-doc/latest/glossary.html#term-loop-lifting "loop lifting"
[dispatcher-inspect-types]: https://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#Dispatcher.inspect_types "dispatch inspect types"
[NUMBA_WARNINGS]: https://numba.pydata.org/numba-doc/latest/reference/envvars.html#envvar-NUMBA_WARNINGS
[NUMBA_DISABLE_JIT]: https://numba.pydata.org/numba-doc/latest/reference/envvars.html#envvar-NUMBA_DISABLE_JIT
[DWARF]: http://www.dwarfstd.org
[NUMBA_GDB_BINARY]: https://numba.pydata.org/numba-doc/latest/reference/envvars.html#envvar-NUMBA_GDB_BINARY
[NUMBA_ENABLE_CUDASIM]: https://numba.pydata.org/numba-doc/latest/reference/envvars.html#envvar-NUMBA_ENABLE_CUDASIM
[CUDA Simulator documentation]: https://numba.pydata.org/numba-doc/latest/cuda/simulator.html#simulator "CUDA 시뮬레이터 문서"

**Note:** 
이 글은 Numba user manual을 번역한 글입니다.
[목차](/dev/numba_user_index)를 보려면 [여기](/dev/numba_user_index)를 클릭하세요.
{: .notice--warning}
