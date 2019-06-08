---
title: "[Numba 사용자 매뉴얼] 1.3 설치"
strapline: "Numba는 파이썬 2.7과 3.5 버전 이후와 호환되고, Numpy 버전은 1.7부터 1.15까지 호환된다."
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

## 호환성

Numba는 파이썬 2.7과 3.5 버전 이후와 호환되고, Numpy 버전은 1.7부터 1.15까지 호환된다.

지원 플랫폼은 다음과 같다:

-   x86 리눅스 (32비트와 64비트)
-   ppcle64 리눅스 (POWER8)
-   윈도우 7 및 그 이후 (32비트와 64비트)
-   OS X 10.9 및 그 이후 (64비트)
-   compute capability 2.0 및 그 이후 버전을 지원하는 NVIDIA GPUs
-   AMD ROC dGPUs (리눅스만 지원하고, AMD Carrizo와 Kaveri APU는 지원 안 함)
-   ARMv7 (Raspberry Pi 2와 3과 같은 32-bit little-endian)

`@jit`을 이용한 [자동 병렬화](/dev/numba_user_parallel)는 64비트 플랫폼에서만 가능하고, 윈도우 파이썬 2.7에서는 지원되지 않는다.

## x86/x86\_64/POWER 플랫폼에 conda를 이용하여 설치하기

Numba를 설치하고 업데이트하는 가장 쉬운 방법은, Anaconda사가 유지하는 크로스 플랫폼 패키지 관리자이자 소프트웨어 배포 프로그램인 `conda`를 사용하는 것이다.
[Anaconda](https://www.anaconda.com/download)를 이용하여 한번에 다 설치하거나 
[Miniconda](https://conda.io/miniconda.html)를 이용하여 conda 환경에 필요한 최소한의 패키지만을 설치할 수 있다.

conda를 설치한 후 아래를 타이핑하자:

    $ conda install numba

또는 아래를 타이핑하자:

    $ conda update numba

Anaconda처럼 Numba도 64비트의 little-endian 모드의 PPC만을 지원함을 주목해야 한다.

Numba의 CUDA GPU 지원을 활성화하기 위해서는 최신의 [그래픽 카드 드라이버](https://www.nvidia.com/Download/index.aspx)를 설치해야 한다.
(대부분의 리눅스 배포판에 기본으로 탑재되는 오픈 소스 Nouveau 드라이버는 CUDA를 지원하지 않음에 유의한다.)
그리고 나서 `cudatoolkit` 패키지를 설치한다.

    $ conda install cudatoolkit

그러므로, NVIDIA에서 CUDA SDK를 직접 설치할 필요가 없다.

## x86/x86\_64 플랫폼에 pip을 사용하여 설치하기

윈도우, 맥, 리눅스를 위한 바이너리 휠 또한 [PyPI](https://pypi.org/project/numba/)에 존재한다.
`pip`을 써서 Numba를 설치할 수 있다:

    $ pip install numba

이것은 Numba에 필요한 부가적인 패키지들을 모두 다운로드받을 것이다. 
필수적인 컴포넌트들은 llvmlite 휠안에 모두 포함되어 있기 때문에 별도로 LLVM을 설치할 필요는 없다
(사실, Numba는 시스템에 설치된 모든 LLVM 버전을 무시한다).

pip으로 설치된 Numba에서 CUDA를 사용하기 위해서는, NVIDIA 사이트에서 [CUDA SDK](https://developer.nvidia.com/cuda-downloads)를 설치할 필요가 있다.
Numba에게 CUDA 라이브러리의 위치를 알려주기 위해서 아래와 같은 환경 변수를 세팅할 필요가 있다:

-   `NUMBAPRO_CUDA_DRIVER` - CUDA 드라이버 공유 라이브러리 파일에 대한 경로
-   `NUMBAPRO_NVVM` - CUDA libNVVM 공유 라이브러리 파일에 대한 경로
-   `NUMBAPRO_LIBDEVICE` - .bc 파일을 포함하고 있는, CUDA libNVVM libdevice *디렉토리*에 대한 경로

## AMD ROCm GPU 지원을 활성화하기

[ROCm 플랫폼](https://rocm.github.io/)은 리눅스상에서 AMD GPU를 활용한 GPU 컴퓨팅을 가능케한다. 
Numba에서 ROCm 지원을 활성화하기 위해서는 conda가 필요한데, Numba 0.40 또는 그 이후 버전과 설치된 Anaconda 또는 Miniconda를 설치하고 나서
다음 순서를 따른다:

1.  [ROCm 설치 명령](https://rocm.github.io/install.html)대로 따른다.
2.  `numba` 채널로부터 `roctools` 콘다 패키지를 설치한다.

        $ conda install -c numba roctools

[roc-examples](https://github.com/numba/roc-examples) 레포지토리에서 샘플 노트북을 찾을 수 있다. 

## 리눅스 ARMv7 플랫폼에 설치하기

[Berryconda](https://https://github.com/jjhelmus/berryconda)는 라즈베리 파이를 위한 콘다 기반 파이썬 배포본이다.
우리는 현재 32비트 little-endian ARMv7 기반 보드를 대상으로 아나콘다 클라우드상의 `numba` 채널에 패키지를 업로드하고 있다.
이것은 라즈베리 파이 2와 3를 지원하지만 1과 Zero는 지원하지 않는다.
`numba` 채널로부터 콘다를 사용하여 설치한다:

    $ conda install -c numba numba

Berryconda와 Numba는 다른 리눅스 기반 ARMv7 시스템에서도 동작 가능하지만 테스트를 해 본적이 없다.

## 소스로부터 설치하기 {#installing-from-source}

다른 파이썬 패키지와 유사하게 소스로부터 설치하는 것은 꽤 쉽지만, 
[llvmlite](https://github.com/numba/llvmlite) 설치는 특별한 LLVM 빌드가 필요하기 때문에 상당히 도전적인 일이 된다.
Numba 개발 목적으로 소스 설치를 하고 있다면 [빌드 환경](http://numba.pydata.org/numba-doc/latest/developer/contributing.html#buildenv)을 참고하기를 바란다.

만약 다른 이유로 소스로부터 실치를 하고 있다면 먼저 [llvmlite 설치 가이드](https://llvmlite.readthedocs.io/en/latest/admin-guide/install.html)를 따른다.
그리고나서 [Github](https://github.com/numba/numba)에서 최신 Numba 소스 코드를 다운로드받는다:

    $ git clone git://github.com/numba/numba.git

최신 버전의 소스 아카이브는 [PyPI](https://pypi.org/project/numba/)에도 있다.
`llvmlite` 외에도 다음과 같은 것도 필요하다:

-   설치된 파이썬과 호환되는 C 컴파일러. Anaconda를 사용하고 있다면, `gcc_linux-64`, `gxx_linux-64` 리눅스 컴파일러 콘다 패키지 또는 
    `clang_osx-64`, `clangxx_osx-64` 맥오에스 콘다 패키지를 설치하면 된다.
-   [NumPy](http://www.numpy.org/)

이제 Numba를 빌드하고 설치할 수 있다:

    $ python setup.py install

## 설치 버전 검사하기

파이썬 프롬프트에서 Numba을 임포트할 수 있어야 한다:

    $ python
    Python 2.7.15 |Anaconda custom (x86_64)| (default, May  1 2018, 18:37:05)
    [GCC 4.2.1 Compatible Clang 4.0.1 (tags/RELEASE_401/final)] on darwin
    Type "help", "copyright", "credits" or "license" for more information.
    >>> import numba
    >>> numba.__version__
    '0.39.0+0.g4e49566.dirty'

또한 numba -s 명령을 실행하여 시스템 능력 등의 정보를 보고 받을 수 있어야 한다:

    $ numba -s
    System info:
    --------------------------------------------------------------------------------
    __Time Stamp__
    2018-08-28 15:46:24.631054

    __Hardware Information__
    Machine                             : x86_64
    CPU Name                            : haswell
    CPU Features                        :
    aes avx avx2 bmi bmi2 cmov cx16 f16c fma fsgsbase lzcnt mmx movbe pclmul popcnt
    rdrnd sse sse2 sse3 sse4.1 sse4.2 ssse3 xsave xsaveopt

    __OS Information__
    Platform                            : Darwin-17.6.0-x86_64-i386-64bit
    Release                             : 17.6.0
    System Name                         : Darwin
    Version                             : Darwin Kernel Version 17.6.0: Tue May  8 15:22:16 PDT 2018; root:xnu-4570.61.1~1/RELEASE_X86_64
    OS specific info                    : 10.13.5   x86_64

    __Python Information__
    Python Compiler                     : GCC 4.2.1 Compatible Clang 4.0.1 (tags/RELEASE_401/final)
    Python Implementation               : CPython
    Python Version                      : 2.7.15
    Python Locale                       : en_US UTF-8

    __LLVM information__
    LLVM version                        : 6.0.0

    __CUDA Information__
    Found 1 CUDA devices
    id 0         GeForce GT 750M                              [SUPPORTED]
                          compute capability: 3.0
                               pci device id: 0
                                  pci bus id: 1

(길이 제약으로 인해 출력이 잘렸음)

**Note:** 
이 글은 Numba user manual을 번역한 글입니다.
[목차](/dev/numba_user_index)를 보려면 [여기](/dev/numba_user_index)를 클릭하세요.
{: .notice--warning}
