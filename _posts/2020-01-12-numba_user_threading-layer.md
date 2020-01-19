---
title: '[Numba 사용자 매뉴얼] 1.14 쓰레딩 레이어'
strapline: "본 절은 쓰레딩 레이어에 관한 것이다."
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
toc: false
comments: true
mathjax: true
last_modified_at: 
---

본 절은 쓰레딩 레이어에 관한 것이다.
쓰레딩 레이어는 CPU 대상으로 `parallel`를 사용하여 내부적으로 병렬 실행을 할 때 사용하는 라이브러리를 뜻한다:

-   `@jit`과 `@njit`에서 `parallel=True` 사용
-   `@vectorize`와 `@guvectorize`에서 `target='parallel'` 사용.

**Note:** 
`threading` 또는 `multiprocessing` 모듈을 사용하지 않더라도 Numba와 함께 오는 기본 쓰레딩 레이어는 잘 동작할 것이고,
부가적인 액션은 필요없다.
{: .notice--warning}



어떤 쓰레딩 레이어가 있는가?
=====================================

아래와 같이 세 종류의 쓰레딩 레이어가 있다:

-   `tbb` - 인텔 TBB에 의해 수행되는 쓰레딩 레이어
-   `omp` - OpenMP에 의해 수행되는 쓰레딩 레이어
-   `workqueue` - 단순 내부 작업-공유 태스크 스케줄러.

실질적으로 현재 보장되는 유일한 쓰레딩 레이어는 `workqueue`이다.
`omp` 레이어는 적절한 OpenMP 실행 라이브러리를 요구한다.
`tbb` 레이어는 Intel TBB 라이브러리가 필요하고 아래처럼 conda 명령을 통해 구할 수 있다:

    $ conda install tbb

`pip`으로 Numba를 설치하였다면 TBB는 다음과 같이 수행하여 받을 수 있다:

    $ pip install tbb

manylinux1 및 다른 이식성 관련한 호환성 이슈로 인해 PyPI에서 배포되는 Numba 바이너리에서
OpenMP 쓰레딩 레이어는 비활성화 상태이다.

**Note:** 
Numba가 쓰레딩 레이어를 찾고 로딩하는 기본 방식은 라이브러리 누락, 비호환 실행모듈 등에 안전하다.
{: .notice--warning}


쓰레딩 레이어 설정 {#numba-threading-layer-setting-mech}
===========================

환경 변수 `NUMBA_THREADING_LAYER`를 통해 또는 `numba.config.THREADING_LAYER`에 값을 할당함으로써
쓰레딩 레이어가 설정된다.
쓰레딩 레이어를 프로그램적으로 설정할 때에는 병렬 로직에 대한 Numba 컴파일이 일어나기 전에 설정되어야 한다.
쓰레딩 레이어를 선택하기 위한 두가지 접근법이 있는데,
첫번째는 다양한 병렬 실행 형태를 고려하는 쓰레딩 레이어를 선택법이고,
두번째는 명시적으로 쓰레딩 레이어 이름(e.g. `tbb`)을 지정함으로써 선택하는 방법이다.

안전한 병렬 실행을 위한 쓰레딩 레이어 선택법
-------------------------------------------------------

병렬 실행은 근본적으로 핵심 파이썬 라이브러리에서 유도된 네 가지 형태로 구성된다:

-   `threading` 모듈의 `threads`
-   `multiprocessing` 모듈로부터 `spawn`을 통해 생겨난 프로세스
    (윈도우에서는 기본이고, 파이썬 3.4 이상에서는 유닉스에서도 지원함)
-   `multiprocessing` 모듈로부터 `fork`를 통해 포킹된 프로세스
   (유닉스에서는 기본)
-   `multiprocessing` 모듈로부터 `forkserver` (유닉스상에서는 파이썬 3에서만 유효함)
    를 통해 포킹된 프로세스.
    새로운 프로세스가 spawn되고 나서 요청에 따라 `fork`가 수행된다.

이런 형태의 병렬화를 사용하는 어떠한 라이브러리라도 주어진 패러다임에서는 안전하게 동작한다.
그 결과, 쓰레딩 레이어 선택 방법은 쉽고, 플랫폼 및 환경 독립적인 방법으로 주어진 패러다임에서 안전한 쓰레딩 레이어를 제공하도록 설계되어 있다.
[numba-threading-layer-setting-mech]에 제공되는 옵션은 다음과 같다:

-   `default`는 특정한 안전성 보증이 없으며 기본이다.
-   `safe`는 fork 및 thread safe하며, 이는 `tbb` 패키지(인텔 TBB 라이브러리)가 설치되어 있어야 한다. 
-   `forksafe`는 fork safe한 라이브러리를 제공한다.
-   `threadsafe`는 thread safe한 라이브러리를 제공한다.

선택된 쓰레딩 레이어를 볼려면 병렬 실행 후 함수 `numba.threading_layer()`를 호출하면 알 수 있다.
예를 들면, TBB가 없는 리눅스 기계의 경우:

```python
    from numba import config, njit, threading_layer
    import numpy as np

    # set the threading layer before any parallel target compilation
    config.THREADING_LAYER = 'threadsafe'

    @njit(parallel=True)
    def foo(a, b):
        return a + b

    x = np.arange(10.)
    y = x.copy()

    # this will force the compilation of the function, select a threading layer
    # and then execute in parallel
    foo(x, y)

    # demonstrate the threading layer chosen
    print("Threading layer chosen: %s" % threading_layer())
```

는 아래 값을 출력한다:

    Threading layer chosen: omp

이 것은 리눅스에 현존하는 GNU OpenMP를 뜻하며 thread safe이다.

이름으로 쓰레딩 레이어 선택하기
---------------------------------

고급 사용자는 특정 쓰레딩 레이어를 직접 선택하기를 원한다.
이 경우 [numba-threading-layer-setting-mech]에 쓰레딩 레이어 이름을 제공하면 된다.
옵션 및 필수 패키지는 다음과 같다:

| Threading Layer   | Platform | Requirements                          |
| Name              |          |                                       |
|----|----|
| `tbb`             | All     | The `tbb` package                     |
|                   |         | (`$ conda install tbb`)               |
|----|----|
| `omp`             | Linux   | GNU OpenMP libraries (very likely     |
|                   |         | this will already exist)              |
|                   |         |                                       |
|                   | Windows | MS OpenMP libraries (very likely this |
|                   |         | will already exist)                   |
|                   |         |                                       |
|                   | OSX     | The `intel-openmp` package            |
|                   |         | (`$ conda install intel-openmp`)      |
|----|----|
| `workqueue`       | All     | None                                  |
|----|----|

쓰레딩 레이어가 잘 로드되지 않는다면 Numba는 문제를 해결하기 위한 힌트를 제공한다.
Numba 진단 명령 `numba -s`는 `__Threading Layer Information__` 절을 가지고 있고
현재 환경에서의 쓰레딩 레이어들의 정보를 보고한다.

부가 설명
===========

쓰레딩 레이어는 CPython 내부와 시스템 라이브러리와 꽤 복잡한 상호작용을 하고 있어서 몇가지 설명거리가 있다:

-   인텔 TBB 라이브러리 설치는 쓰레딩 레이어 선택에 있어서 많은 옵션을 제공해 준다.
-   리눅스상에서 `omp` 쓰레딩 레이어는 fork safe 하지 않는다 (GNU OpenMP 실행시간 라이브러리 `libgomp`가 fork safe하지 않기 때문에).
    `omp` 쓰레딩 레이어를 사용하는 프로그램에서 fork가 일어날 때 자동으로 fork 액션을 발견하여
    조용히 포킹된 자식 프로세스를 종료시키고 `STDERR`에 에러 메시지를 출력한다.
-   OSX상에서 OpenMP 기반 쓰레딩 레이어를 활성화시키기 위해서는  `intel-openmp` 패키지가 필요하다.
-   파이썬 2.7을 운용하는 윈도우 사용자의 경우 `tbb` 쓰레딩 레이어는 사용할 수 없다.


[numba-threading-layer-setting-mech]: #numba-threading-layer-setting-mech "쓰레딩 설정"

**Note:** 
이 글은 Numba user manual을 번역한 글입니다.
[목차](/dev/numba_user_index)를 보려면 [여기](/dev/numba_user_index)를 클릭하세요.
{: .notice--warning}
