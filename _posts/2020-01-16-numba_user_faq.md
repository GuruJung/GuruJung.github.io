---
title: '[Numba 사용자 매뉴얼] 1.16 자주하는 질문'
strapline: "컴파일되는 함수에 함수를 인수로 전달할 수 있는가?"
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

프로그래밍
===========

컴파일되는 함수에 인수로서 함수를 전달할 수 있는가?
----------------------------------------------------------

Numba 0.39부터 함수 인수도 JIT 컴파일된 함수라면 가능하다:
As of Numba 0.39, you can, so long as the function argument has also
been JIT-compiled:

```python
    @jit(nopython=True)
    def f(g, x):
        return g(x) + g(-x)

    result = f(jitted_g_function, 1)
```

그러나 함수 인수를 처리하는 데에는 별개의 오버헤드가 드는데,
이게 응용프로그램에 부담을 준다면 클로저에서 함수 인수를 캡쳐하는 팩토리 함수를 사용할 수도 있다:

```python
    @jit(nopython=True)
    def make_f(g):
        # Note: a new f() is created each time make_f() is called!
        @jit(nopython=True)
        def f(x):
            return g(x) + g(-x)
        return f

    f = make_f(jitted_g_function)
    result = f(1)
```

Numba에서 함수의 디스패치 성능을 향상시키는 일은 계속 작업중이다.
Improving the dispatch performance of functions in Numba is an ongoing
task.

전역 변수를 고칠 때 Numba는 그걸 잘 모르는 것 같다.
-----------------------------------------------------------

Numba는 전역 변수를 컴파일-시간 상수로 간주한다.
전역 변수의 값이 수정될 때 그것을 반영하는 컴파일된 함수를 원한다면,
[Dispatcher.recompile] 메쏘드를 통해 다시 컴파일해야 한다.
이 방법은 상대적으로 느린 연산이어서, 원래 코드를 리팩토링하고 전역 변수를 함수 인수로 
전달하는 편이 더 낫다.

컴파일된 함수를 디버그할 수 있는가?
------------------------------

[pdb]나 다른 비슷한 고수준 도구를 이용하는 것은 현재 지원되지 않는다.
다만 [NUMBA_DISABLE_JIT] 환경 변수를 설정함으로써 임시로 컴파일을 막는 것은 가능한다.

포트란-순서로 된 배열을 생성할 수 있는가?
-----------------------------------------

Numba는 대부분의 Numpy 함수들이 지원하는 `order` 인수를 지원하지 않는다
([type inference] 알고리즘의 제약때문에).
C-순서 배열을 생성하고 트랜스포즈 함으로써 이런 이슈를 우회할 수 있다.
예를 들어:

```python
    a = np.empty((3, 5), order='F')
    b = np.zeros(some_shape, order='F')
```

위 코드는 아래 코드로 재작성될 수 있다:

```python
    a = np.empty((5, 3)).T
    b = np.zeros(some_shape[::-1]).T
```

정수 범위를 늘릴 수 있는 방법이 있는가?
---------------------------------

기본적으로 Numba는 일반적인 컴퓨터 기계에서 정한 정수를 사용한다.
그런데 32비트 기계에서 64비트 정수를 쓰고자 한다면,
관련 변수를 `np.int64` (예를 들어, `0` 대신에 `np.int64(0)`)로 초기화하면 된다.
그러면 관련 변수들은 모두 계산시에 그렇게 전파될 것이다.

`parallel=True`가 작동하고 있는지 어떻게 알수 있는가? {#parallel_faqs}
-----------------------------------------

환경 변수 [NUMBA_WARNINGS]을 0이 아닌 수로 세팅하면,
`parallel=True` 변환이 실패할 때 경고가 표시될 것이다.

환경 변수 [NUMBA_DEBUG_ARRAY_OPT_STATS]를 세팅하면 병렬 for 루프에 대한 통계를 제공해 준다.

성능
===========

Numba는 함수를 인라인시키는가?
----------------------------

Numba는 LLVM에 충분한 정보를 주기 때문에 충분히 짧은 함수들은 인라인화될 수 있다.
이 기능은 [nopython mode]에서만 작동한다.

Numba는 배열 연산시 벡터라이즈할 수 있는가 (SIMD)?
-----------------------------------------------

Numba는 스스로는 그런 최적화를 하지 않지만, LLVM이 그런 최적화를 할 수 있도록 지시한다.

왜 루프가 벡터라이즈되지 않을까요?
------------------------------

기본적으로 LLVM에서의 루프-벡터화 최적화가 적용되는데,
그것이 강력한 최적화임에도 불구하고 모든 루프에 적용 가능한 것은 아니다.
때때로, 루프-벡터화는 메모리 접근 패턴 등의 미묘한 사항으로 인해 실패하기도 한다.
부가적인 진단 정보를 보기 위해서는 아래와 같은 줄을 추가하자:

```python
import llvmlite.binding as llvm
llvm.set_option('', '--debug-only=loop-vectorize')
```

이는 LLVM이 **루프-벡터화** 과정에서 발생한 디버그 정보를 stderr에 출력하도록 한다.
각각의 함수 항목은 다음처럼 보인다:

```
LV: Checking a loop in "<low-level symbol name>" from <function name>
LV: Loop hints: force=? width=0 unroll=0
...
LV: Vectorization is possible but not beneficial.
LV: Interleaving is not beneficial.
```

각각의 함수 항목은 빈 라인으로 구별이 되는데, 벡터화를 거부하는 이유는 주로 항목의 끝에 나와 있다.
위의 예에서는 벡터화가 도움이 되지 않기 때문에 LLVM 거부했음을 알 수 있다.
이 경우 원인은 메모리 접근 패턴에 기인할 수있는 데,
예를 들면 루핑이 되는 배열이 연속으로 되어 있는 배치가 아니기 때문일수도 있다.

메모리 영역을 결정할 수 없는 부분에서는 메모리 접근 패턴이 사소하지 않기 때문에
LLVM이 다음과 같은 메시지를 뿌리면서 거부할 수도 있다:

```
LV: Can't vectorize due to memory conflicts
```

또다른 주요 원인은:

```
LV: Not vectorizing: loop did not meet vectorization requirements.
```

이 경우 벡터화된 코드가 다르게 행동하기 때문에 거부되는 이유이다.
fastmath 명령을 허용하기 위해서 `fastmath=True`를 해보는 것도 한가지 해결책이다.

Numba는 자동으로 코드를 병렬화하는가?
------------------------------------------

몇몀 경우에 할 수 있다:

-   `target="parallel"` 옵션이 켜진 ufunc과 gufunc의 경우 복수의 쓰레드상에서 실행될 것이다.
-   `@jit`의 `parallel=True` 옵션은 배열 연산을 최적화하고 병렬로 실행할 것이다.
     명시적으로 루프를 병렬화하기 위한 `prange()` 지원도 있다.


`nogil=True` 옵션([releasing the GIL] 참조)을 쓰면서 수동으로 복수의 쓰레드상에 계산을 수행할 수도 있다.
또한 CUDA나 HSA 백엔드로 GPU상에서 병렬 실행을 수행하는 수도 있다.

단길이 함수를 스피드업할 수 있는가?
-------------------------------------------

별로 중요치 않다. 
때때로 새로운 사용자는 아래와 같은 함수를 JIT 컴파일할려고 한다:

```python
    def f(x, y):
        return x + y
```

그리고 파이썬 해석기보다 훨씬 큰 스피드업을 기대한다.
그러나 Numba가 여기에서 개선시킬 수 있는 게 많지 않다;
시간의 대부분은 아마도 CPython 함수 호출 메커니즘에서 소비하고,
함수 그 자체는 별로 쓰지않는다.
일반적으로 함수가 10 µs (역주: 마이크로초)이하로 걸린다면 그대로 두는게 낫다.

예외는 또다른 컴파일된 함수로부터 호출되는 함수는 컴파일해 주어야 한다.

복잡한 함수를 컴파일할 때 지연이 생기는데 어떻게 개선시킬 수 있는가?
---------------------------------------------------------------------------------

`@jit` 데코레이터에 `cache=True`를 전달하면,
나중 사용을 위해 디스크에 컴파일된 버전을 유지할 것이다.

좀더 급진적인 대안은 [ahead-of-time compilation]이다.

GPU 프로그래밍
===============

`CUDA intialized before forking` 에러를 어떻게 피해갈 수 있는가?
----------------------------------------------------------------

리눅스상에서 파이썬 표준 라이브러리인 `multiprocessing` 모듈은 기본으로
새로운 프로세스를 생성하기 위해서 `fork` 메쏘드를 쓴다.
프로세스 포킹은 부모와 자식 프로세스간의 상태를 중첩시키기 때문에
CUDA 런타임이 포크*전에* 초기화되었다면 자식 프로세스에서 CUDA가 제대로 동작하지 못 할 것이다.
Numba는 이런 상황을 발견하면 `CUDA initialized before forking` 메시지와 함께
`CudaDriverError` 예외를 던진다.

본 에러를 피하기 위해서 `numba.cuda` 함수를 프로세스 풀이 다 생성된 이후 또는 자식 프로세스안에서만
호출하는 방법이 있다.
예를 들어 프로세스 풀을 가동하기 전에 이용가능한 GPU 개수를 질의하는 경우처럼
이 건은 항상 가능한 경우는 아니다.
파이썬3에서는 [multiprocessing documentation](https://docs.python.org/3.6/library/multiprocessing.html#contexts-and-start-methods)에서 언급되는 것처럼 프로세스 시작 방법을 바꿀 수 있다.
즉, `fork`를 `spawn`이나 `forkserver`로 바꾸면 CUDA 초기화 이슈를 피해갈 수 있다.
다만 자식 프로세스가 그들 부모로부터 어떠한 전역 변수도 상속받을 수는 없을 것이다.


다른 유틸리티와의 통합
================================

Numba를 사용한 애플리케이션을 "freeze"할 수 있나요?
-------------------------------------------------

당신이 PyInstaller나 유사 유틸리티를 사용하여 애플리케이션을 freeze하려 한다면
llvmlite와 관련된 이슈를 만나게 될 수 있다.
llvmlite는 non-python dll을 가지고 있는데 
보통 `llvmlite/binding/libllvmlite.so`이나 `llvmlite/binding/llvmlite.dll` 이다.
그런데 freeze 유틸리티가 그것을 자동으로 발견 못 하는 경우가 있으므로
freeze 유틸리티에게 해당 DLL의 위치를 알려주어야 한다.

스파이더에서 두번 스크립트를 호출할 때 에러가 난다.
-----------------------------------------------------

스파이더의 콘솔에서 스크립트를 실행할 때
우선 스파이더는 기존 모듈을 재로드할려고 한다.
이 행동은 Numba와는 잘 맞지 않아서
`TypeError: No matching definition for argument type(s)`와 같은 에러를 유발한다.

스파이더 환경 설정에 방법이 있다.
`Preferences` 윈도우를 열고, `Console`를 선택한 후 `Advanced Settings`에서 `Set UMR excluded modules` 버튼을 클릭하고
`numba`를 추가한다.

세팅이 효과가 있기 위해서는 IPython 콘솔이나 커널을 재시작하여야 한다.


Numba가 왜 현재 로케일에 대해서 불평하는가? {#llvm-locale-bug}
-------------------------------------------------

만약 다음과 같은 에러 메시지를 만나게 된다면:

```
    RuntimeError: Failed at nopython (nopython mode backend)
    LLVM will produce incorrect floating-point code in the current locale
```

실수 상수를 잘못 다루는 LLVM 버그에 마주치게 된 것이다.
이 현상은 matplotlib에 Qt 백엔드같은 특정한 제3자 라이브러리를 사용하게 될 때 일어난다고 알려져 있다.

버그를 회피하기 위해서는 로케일을 기본값으로 강제 조정할 필요가 있다.
예를 들면:

```python
    import locale
    locale.setlocale(locale.LC_NUMERIC, 'C')
```

기타
=============

다른 작업물에서 Numba를 참조/인용/acknowledge를 어떻게 해야 하는가?
--------------------------------------------------------

학술적 이용에 관해서는 우리의 ACM 논문을 참조하는 것이 제일 좋다:
[Numba: a LLVM-based Python JIT
compiler.](http://dl.acm.org/citation.cfm?id=2833162&dl=ACM&coll=DL)


[Dispatcher.recompile]: https://numba.pydata.org/numba-doc/0.42.0/reference/jit-compilation.html#Dispatcher.recompile
[pdb]: https://docs.python.org/3/library/pdb.html#module-pdb
[NUMBA_DISABLE_JIT]: https://numba.pydata.org/numba-doc/latest/reference/envvars.html#envvar-NUMBA_DISABLE_JIT
[type inference]: https://numba.pydata.org/numba-doc/latest/glossary.html#term-type-inference "타입 추론"
[NUMBA_WARNINGS]: https://numba.pydata.org/numba-doc/latest/reference/envvars.html#envvar-NUMBA_WARNINGS
[nopython mode]: https://numba.pydata.org/numba-doc/latest/glossary.html#term-nopython-mode "nopython 모드"
[releasing the GIL]: /dev/numba_user_jit#jit-nogil "GIL 해제하기"
[ahead-of-time compilation]: /dev/numba_user_pycc#pycc
