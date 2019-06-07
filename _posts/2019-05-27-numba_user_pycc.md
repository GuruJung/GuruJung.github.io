---
title: "[Numba 사용자 매뉴얼] 1.9 조기 코드 컴파일"
strapline: "Numba의 주 사용 케이스는 적기(Just-in-Time) 컴파일이지만 조기(Ahead-of-Time) 컴파일(AOT)도 지원한다."
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

Numba의 주 사용 케이스는 [적기(Just-in-Time) 컴파일](http://numba.pydata.org/numba-doc/latest/glossary.html#term-just-in-time-compilation)이지만
[조기(Ahead-of-Time) 컴파일](http://numba.pydata.org/numba-doc/latest/glossary.html#term-ahead-of-time-compilation)(AOT)도 지원한다.

## 개요 

### 장점

1.  AOT 컴파일은 Numba에 의존하지 않는 컴파일된 확장 모듈을 생성해낸다:
    Numba가 설치되지 않은 기계에도 해당 모듈을 배포할 수 있다 (다만 Numpy는 그래도 필요하다).  
2.  실행시에 컴파일 관련 오버헤드가 없다 (`@jit` [cache](/dev/numba_user_jit#jit-cache) 옵션도 참조하자).
    또한 Numba를 임포트하는 비용도 없다.

**See also:** 
[파이썬 패키징 사용자 가이드](https://packaging.python.org/en/latest/extensions/)에서 컴파일된 확장 모듈에 대해서 논의하고 있다.
{: .notice--info}

### 제약점

1.  AOT 컴파일은 정규 함수에 대해서만 가능하고, [ufuncs](http://numba.pydata.org/numba-doc/latest/glossary.html#term-ufunc)은 컴파일하지 못 한다.
2.  함수 시그너처를 명시적으로 지정해야 한다.
3.  익스포트된 함수 각각은 단지 하나의 시그너처만을 가질 수 있다. 그러나, 같은 로직에 대해서 다른 이름으로 다른 시그너처를 익스포트할 수 있다.
4.  AOT 컴파일은 CPU 아키텍처 패밀리(e.g. "x86-64")에 대해서 모두 잘 작동하는 보편적 코드를 생산하는 반면에 
    JIT 컴파일은 현재 CPU에 최적화된 코드를 생성한다.

## 사용법

### 독립 예제

```python
    from numba.pycc import CC

    cc = CC('my_module')
    # Uncomment the following line to print out the compilation steps
    #cc.verbose = True

    @cc.export('multf', 'f8(f8, f8)')
    @cc.export('multi', 'i4(i4, i4)')
    def mult(a, b):
        return a * b

    @cc.export('square', 'f8(f8)')
    def square(a):
        return a ** 2

    if __name__ == "__main__":
        cc.compile()
```

위 파이썬 스크립트를 실행하면 `my_module`이라는 확장 모듈을 생성한다.
현재 플랫폼에 따라 실제 파일 이름은 `my_module.so`, `my_module.pyd`, `my_module.cpython-34m.so` 등이 될 수 있다.

생성된 모듈은 세 개의 함수를 가진다: `multf`, `multi`와 `square`.
`multi`는 32비트 정수(`i4`)상에 작동하는 반면에
`multf`와 `square`는 배정도 실수(`f8`에 대해서 작동한다.

```python
    >>> import my_module
    >>> my_module.multi(3, 4)
    12
    >>> my_module.square(1.414)
    1.9993959999999997
```

### Distutils 통합

distutils 또는 setuptools를 사용할 때 `setup.py` 스크립트 파일에 확장 모듈에 대한 컴파일 단계를 통합시키는 방법도 있다:

```python
    from distutils.core import setup

    from source_module import cc

    setup(...,
          ext_modules=[cc.distutils_extension()])
```

위의 `source_module`는 `cc` 개체를 정의하는 모듈이다.
이처럼 컴파일되는 확장 모듈은 당신의 파이썬 프로젝트의 빌드 파일에 자동으로 포함된다.
그래서 wheel이나 Conda 패키지같은 바이너리 패키지 내에 확장 모듈을 배포할 수 있다.
conda를 사용하는 경우에 AOT 컴파일러는 Anaconda 배포물에 포함된 컴파일러여야 한다. 

### 시그너처 문법

익스포트되는 시그너처의 문법은 `@jit` 데코레이터와 같다.
[types](http://numba.pydata.org/numba-doc/latest/reference/types.html#numba-types) 레퍼런스에서 해당 항목을 더 읽어 보도록 하자.

여기에 1d 배열상의 second-order centered difference 함수를 익스포팅하는 예제이다:

```python
    @cc.export('centdiff_1d', 'f8[:](f8[:], f8)')
    def centdiff_1d(u, dx):
        D = np.empty_like(u)
        D[0] = 0
        D[-1] = 0
        for i in range(1, len(D) - 1):
            D[i] = (u[i+1] - 2 * u[i] + u[i-1]) / dx**2
        return D
```

또한 리턴 타입을 빼먹는다면 Numba가 대신 추정해 줄 것이다.

```python
    @cc.export('centdiff_1d', '(f8[:], f8)')
    def centdiff_1d(u, dx):
        # Same code as above
        ...
```
