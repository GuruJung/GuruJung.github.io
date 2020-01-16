---
title: '[Numba 사용자 매뉴얼] 1.12 JIT 코드에서 파이썬 콜백 호출'
strapline: "Numba가 컴파일할 수 없는 어떤 콜백 코드를 nopython 모드에서 호출해야만 하는 경우가 있다."
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

Numba가 컴파일할 수 없는 어떤 콜백 코드를 nopython 모드에서 호출해야만 하는 경우가 있다.
바로 아래와 같은 경우이다:

-   오랫동안 실행되는 JIT 함수를 위한 진행 로깅
-   Numba가 지원하지 못 하는 데이터 구조를 사용함
-   파이썬 디버거를 사용해 JIT 코드 내부를 디버깅하기.

Numba가 파이썬 콜백을 호출할 때 아래와 같은 일이 벌어진다:

-   GIL을 획득한다.
-   native한 표현값을 파이썬 개체로 변환한다.
-   파이썬 인터프리터에서 콜백을 호출한다.
-   콜백에서 반환된 값을 native 표현값으로 변환한다.
-   GIL을 푼다.

이런 단계는 비싼 비용이 들기 때문에 성능이 중요한 부분에서는 여기에 언급된 기능을 사용해서는 **안** 된다.

`objmode` 문맥 관리자 {#with_objmode}
=============================

**Warning:** 
본 기능은 오용되기 쉽다.
본 기능을 사용하기 전에 다른 대안이 있는지 먼저 고려해야 한다.
{: .notice--warning}

JIT 코드안에서 Numba.objmode(*args, **kwargs)를 사용하여 문맥 관리자를 생성함으로써
파이썬 해석기를 사용하기 위한 object 모드로 들어갈 수 있다.
이런 변환 프로세스는 제약이 있어서 모든 파이썬 코드를 처리할 수는 없지만, 사용자는 복잡한 로직을 또다른 파이썬 함수로 작성한 다음에
해당 파이썬 함수를 실행하는 식으로 피해갈 수 있다.

키워드 인수 방식으로만 본 함수를 사용하라.
인수 이름은 with 블럭 바깥의 출력 변수에 대응되어야 한다.
인수 값은 타입을 표현하는 문자열이다.
with 블럭에서 벗어날 때 출력 변수는 해당 타입으로 변환된다.
본 처리 방식은 nopython 함수의 인수로 파이썬 개체를 넘길 때와 똑같은 것이다.

예제:
```python
import numpy as np
from numba import njit, objmode

def bar(x):
    # This code is executed by the interpreter.
    return np.asarray(list(reversed(x.tolist())))

@njit
def foo():
    x = np.arange(5)
    y = np.zeros_like(x)
    with objmode(y='intp[:]'):  # annotate return type
        # this region is executed by object-mode.
        y += bar(x)
    return y
```

Note
Known limitations:
with-block cannot use incoming list objects.
with-block cannot use incoming function objects.
with-block cannot yield, break, return or raise such that the execution will leave the with-block immediately.
with-block cannot contain with statements.
random number generator states do not synchronize; i.e. nopython-mode and object-mode uses different RNG states.

**Note:** 
알려진 제약사항:

-   with 블럭은 인입 리스트 개체를 사용할 수 없다.
-   with 블럭은 인입 함수 개체를 사용할 수 없다.
-   with 블럭내에서 블럭을 즉시 빠져 나가게 하는 yield, break, return, raise 키워드를 사용할 수 없다.
-   랜덤 넘버 생성기의 상태는 동기화되지 않는다; 즉, nopython 모드와 object 모드는 서로 다른 상태를 사용한다.
{: .notice--warning}

**Note:** 
문맥 관리자가 nopython 모드 바깥에서 사용될 때는 아무 일도 하지 않는다.
{: .notice--warning}

**Warning:** 
본 기능은 실험적이다. 별도의 공지 없이 기능이 변경될 수 있다.
{: .notice--warning}

