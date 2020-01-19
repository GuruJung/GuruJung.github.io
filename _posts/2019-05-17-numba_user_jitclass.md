---
title: "[Numba 사용자 매뉴얼] 1.7 @jitclass로 파이썬 클래스 컴파일하기"
strapline: "Numba는 numba.jitclass 데코레이터를 통해 클래스에 대한 코드 생성을 지원한다."
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

**Note:** jitlcass 지원은 현재 초기 상태이다. 모든 특징들이 구현된 것은 아니다.
{: .notice--warning}

jitclass 데코레이터를 통해 클래스에 대한 코드 생성을 지원한다. 
각 필드에 대한 타입 정의와 함께 본 데코레이터를 사용하면 주어진 클래스는 최적화되도록 마킹된다.
이렇게 만들어진 클래스를 우리는 jitclass라고 부른다.
jitclass의 모든 함수가 nopython 함수로 컴파일된다. 
jitclass 인스턴스의 데이터는 C 호환 구조로서 힙 상에 할당된다.
그래서 컴파일된 어떤 함수에서든지 인터프리터를 통하지 않고 해당 데이터에 직접 접근할 수 있다.

## 기본 사용법

jitclass의 예제 하나를 보자:

```python
    import numpy as np
    from numba import jitclass          # import the decorator
    from numba import int32, float32    # import the types

    spec = [
        ('value', int32),               # a simple scalar field
        ('array', float32[:]),          # an array field
    ]

    @jitclass(spec)
    class Bag(object):
        def __init__(self, value):
            self.value = value
            self.array = np.zeros(value, dtype=np.float32)

        @property
        def size(self):
            return self.array.size

        def increment(self, val):
            for i in range(self.size):
                self.array[i] = val
            return self.array
```

(소스 트리내의 examples/jitclass.py에서 전체 예제를 보자)

위의 예에서 `spec`은 2-튜플의 리스트로서 제공되는 데, 튜플은 필드 이름과 필드의 타입을 포함하고 있다.
이 것 대신에 사용자는 필드 이름을 타입에 매핑하는 사전(안정적인 필드 순서를 유지하기 위해 `OrderedDict`를 선호)을 사용할 수도 있다.

클래스 정의에는 최소한 필드들을 초기화하는 `__init__` 메쏘드가 필요하다.
초기화되지 않은 필드가 있다면 해당 필드는 쓰레기 데이터를 가지고 있게 된다.
메쏘드와 속성(getter와 setter)이 정의될 수 있고 자동으로 컴파일될 것이다. 


## 지원되는 연산

jitclass의 연산들중에 아래 것들은 인터프리터와 numba로 컴파일된 함수 모두에서 잘 작동한다:

-   새 인스턴스를 생성하기 위해서 jitclass 클래스 개체를 호출하기 (e.g. `mybag = Bag(123)`)
-   멤버 변수(attribute)와 속성(roperty)에 대한 읽기/쓰기 접근 (e.g. `mybag.value`)
-   메쏘드 호출 (e.g. `mybag.increment(3)`)

컴파일된 함수안에서 jitclass를 사용하는 것이 훨씬 효율적이다.
짧은 메쏘드는 LLVM 인라이너에 의해서 인라인될 수 있다.
멤버 변수는 단순히 C 구조로부터 바로 읽힐 수 있다.
인터프리터에서 jitclass를 사용하는 것은 인터프리터에서 numba-컴파일된 함수를 호출하는 것과 같은
정도의 오버헤드가 있다. 
인수와 리턴값은 파이썬 개체와 원래의 표현(native representation) 사이에 언박싱 또는 박싱이 되어야 한다.
jitclass 인스턴스가 인터프리터에서 다루어지게 될 때 jitclass의 값들은 파이썬 개체로 박싱이 되지 않는다.
멤버를 통한 필드 값 접근 때에만 박싱이 이루어진다.

## 제약 사항

-   numba-컴파일된 함수안에서 jitclass 클래스 개체는 함수(생성자)로 취급된다.
-   `isinstance()`는 인터프리터에서만 작동된다.
-   인터프리터상에서 jitclass 인스턴스를 다루는 것은 아직 최적화되어 있지 않다.
-   jitclass는 CPU상에서만 지원된다. (추후 GPU에 대해서도 지원될 것이다.)


**Note:** 
이 글은 Numba user manual을 번역한 글입니다.
[목차](/dev/numba_user_index)를 보려면 [여기](/dev/numba_user_index)를 클릭하세요.
{: .notice--warning}
