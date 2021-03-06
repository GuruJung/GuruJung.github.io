---
title: "[Numba 사용자 매뉴얼] 1.2 개요"
strapline: 
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

Numba는 파이썬 배열 및 수치 함수를 위한 컴파일러으로서 파이썬 코드로 고성능 애플리케이션을 작성할 수 있게 해준다.

Numba는 [LLVM 컴파일러 인프라구조](http://llvm.org/)를 이용하여 순수 파이썬 코드를 최적화된 기계어로 변환한다. 
배열과 수학식이 많이 쓰인 파이썬 코드에 몇 가지 주석을 닮으로써 언어나 파이썬 해석기를 교체하지 않고도 
C, C++, 포트란과 비슷한 성능까지 파이썬 코드를 그때 그때 실시간으로 최적화할 수 있다.

Numba의 주요 특징은 아래와 같다:

-   [즉석 코드 생성](/dev/numba_user_jit) (임포트나 실행시 또는 사용자가 원하는 시점에)
-   CPU (기본값) 또는 [GPU](http://numba.pydata.org/numba-doc/latest/cuda/index.html)용 기계어 코드 생성
-   (Numpy를 통한) 파이썬 과학 소프트웨어와의 통합

인수로 Numpy 배열을 취하는 Numba-최적화된 함수의 예를 보자:

```python
    @numba.jit
    def sum2d(arr):
        M, N = arr.shape
        result = 0.0
        for i in range(M):
            for j in range(N):
                result += arr[i,j]
        return result
```

**Note:** 
이 글은 Numba user manual을 번역한 글입니다.
[목차](/dev/numba_user_index)를 보려면 [여기](/dev/numba_user_index)를 클릭하세요.
{: .notice--warning}
