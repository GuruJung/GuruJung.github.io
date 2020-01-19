---
title: '[Numba 사용자 매뉴얼] 1.5 `@generated_jit`으로 유연하게 전문화하기'
strapline: "generated_jit 데코레이터는 사용자가 컴파일 시점에 특정 구현의 선택을 제어할 수 있게 해주면서도 JIT 함수의 빠른 실행시간 속도를 그대로 유지시킨다."
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

[numba.jit](https://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#numba.jit) 데코레이터는 많은 경우에 유용하지만,
가끔씩 입력 타입에 따라 구현이 다른 함수가 필요할 때가 있다.
[numba.generated_jit](https://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#numba.generated_jit) 데코레이터는 사용자가 컴파일 시점에 
특정 구현의 선택을 제어할 수 있게 해주면서도 JIT 함수의 빠른 실행시간 속도를 그대로 유지시킨다.

## 예제

주어진 값이 \"누락\" 값인지의 여부를 특정 조건에 따라 판별하는 함수를 작성하고 있다고 가정하자.
예를 들면, 아래와 같은 조건을 채택하자.
-   실수 인수에 대해서는 누락 값이 `NaN`이다.
-   Numpy datetime64와 timedelta64 인수의 경우, 누락 값은 `NaT`이다.
-   이 외의 타입들은 누락 값이라는 게 없다.


[numba.generated_jit](https://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#numba.generated_jit)
데코레이터를 사용하여 이런 컴파일-시간 로직을 쉽게 구현할 수 있다:

```python
    import numpy as np

    from numba import generated_jit, types

    @generated_jit(nopython=True)
    def is_missing(x):
        """
        Return True if the value is missing, False otherwise.
        """
        if isinstance(x, types.Float):
            return lambda x: np.isnan(x)
        elif isinstance(x, (types.NPDatetime, types.NPTimedelta)):
            # The corresponding Not-a-Time value
            missing = x('NaT')
            return lambda x: x == missing
        else:
            return lambda x: False
```

여기에서 몇가지 언급할 것이 있다:

-   데코레이트되는 함수는 인수 값이 아닌, 인수의 
    [Numba 타입](https://numba.pydata.org/numba-doc/latest/reference/types.html#numba-types)으로 호출된다.
-   데코레이트되는 함수는 결과를 실제로 계산하는 것이 아니고, 주어진 타입에 대한 실제 정의를 구현한 호출 가능한 개체를 반환한다.
-   컴파일된 구현 코드안에서 재사용될 수 있도록 컴파일 시간에 어떤 데이터 (위의 경우 `missing` 변수)를 미리 계산하는 것이 가능하다.
-   함수 정의시 인수는 데코레이트되는 함수에서 정의된 것과 같은 이름을 사용해야 기대한 대로 동작한다.
    (역자주: 위 코드에서 lambda 함수의 인수명이 반드시 `x`여야 한다는 의미이다.)

## 컴파일 옵션

[numba.generated\_jit](https://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#numba.generated_jit)
데코레이터는 
[numba.jit](https://numba.pydata.org/numba-doc/latest/reference/jit-compilation.html#numba.jit) 데코레이터와 동일한 키워드 인수를 지원한다.
예를 들어, `nopython`과 `cache` 옵션을 지원한다.


**Note:** 
이 글은 Numba user manual을 번역한 글입니다.
[목차](/dev/numba_user_index)를 보려면 [여기](/dev/numba_user_index)를 클릭하세요.
{: .notice--warning}
