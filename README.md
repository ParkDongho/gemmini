<p align="center">
  <img width="1000" src="./img/full-logo.svg">
</p> 

Gemmini
====================================

The Gemmini project는 전체 시스템 및 전체 스택 DNN 하드웨어 탐색 및 평가 플랫폼을 개발 중입니다.
Gemmini는 아키텍트가 시스템 및 소프트웨어 스택의 다양한 구성 요소(가속기 자체 외부)가 상호작용하여 전체 DNN 성능에 미치는 영향을 유용하게 분석할 수 있도록 합니다.

Gemmini는 [Chipyard](https://github.com/ucb-bar/chipyard) 생태계의 일부이며, [Chisel](https://www.chisel-lang.org/) 하드웨어 설명 언어를 사용하여 개발되었습니다.

이 문서는 Gemmini를 처음 시도하려는 초보자들에게 정보를 제공하고, Gemmini의 소스 코드를 해킹하려는 사람들을 위한 더 깊이 있는 정보를 제공하기 위한 것입니다.

![Gemmini의 고수준 아키텍처](./img/gemmini-system.png)

Quick Start
==========

여기에서는 Gemmini의 종속성(Chipyard 및 Spike)을 설치하고, Gemmini 하드웨어 및 소프트웨어를 빌드한 다음, 해당 소프트웨어를 하드웨어 시뮬레이터에서 실행하는 빠른 가이드를 제공합니다.

Dependencies
---------

시작하기 전에 [Chipyard 종속성](https://chipyard.readthedocs.io/en/latest/Chipyard-Basics/Initial-Repo-Setup.html#default-requirements-installation)을 설치하십시오.

Installing Chipyard and Spike
-----------------------------

다음 단계들을 실행하여 Chipyard와 Spike를 설치하세요(아래에 표시된 올바른 Chipyard 및 Spike 커밋을 체크아웃하는 것을 잊지 마십시오):

```shell
git clone https://github.com/ucb-bar/chipyard.git
cd chipyard
git checkout 1.9.1
./build-setup.sh riscv-tools

source env.sh

cd generators/gemmini
git config remote.origin.fetch "+refs/heads/*:refs/remotes/origin/*"
git fetch && git checkout v0.7.1
git submodule update --init --recursive

make -C software/libgemmini install

# 마지막 단계는 현실적인 DRAM 모델이 있는 MIDAS 시뮬레이션을 실행하려는 경우에만 필요합니다.
cd -
cd sims/firesim
source sourceme-f1-manager.sh --skip-ssh-setup # 이 명령에서 발생하는 오류 메시지를 무시하십시오.
./build-setup.sh --library --skip-validate
```

Setting Up Gemmini
------------------

아래 단계들을 실행하여 Gemmini 구성 파일, 심볼릭 링크 및 하위 디렉터리를 설정하십시오:

```shell
cd chipyard/generators/gemmini
./scripts/setup-paths.sh
```

Building Gemmini Software
-------------------------

아래 단계들을 실행하여 Gemmini 프로그램, ResNet50과 같은 대규모 DNN 모델 및 작은 행렬 곱셈 테스트를 컴파일하십시오.

```shell
cd chipyard/generators/gemmini/software/gemmini-rocc-tests
./build.sh
```

이후, `build/` 디렉터리에서 "baremetal" 환경, Linux 환경 및 "proxy-kernel" 환경용 RISC-V 바이너리를 찾을 수 있습니다.

Linux 바이너리는 Linux를 실행하는 SoC에서 실행하도록 설계되었습니다.
이 바이너리들은 동적으로 링크되며 모든 시스템 호출을 지원합니다.
대부분의 사용자는 이들을 [FireSim](https://fires.im/) 시뮬레이터에서 실행합니다.

Baremetal 바이너리는 운영 체제가 없는 환경에서 실행하도록 설계되었습니다.
이들은 대부분의 시스템 호출을 지원하지 않으며 가상 메모리도 지원하지 않습니다.
대부분의 사용자는 이들을 Verilator 또는 VCS와 같은 사이클 정확도 시뮬레이터에서 실행합니다.

"Proxy-kernel" 바이너리는 ["RISC-V Proxy Kernel"](https://github.com/riscv-software-src/riscv-pk)이라는 축소된 버전의 Linux에서 실행되도록 설계되었습니다.
이 바이너리들은 가상 메모리를 지원하며, 보통 Verilator와 같은 사이클 정확도 시뮬레이터에서 실행됩니다.

**경고:** Proxy-kernel 바이너리는 힙 공간이 제한되어 있으므로 baremetal 또는 Linux 환경에서는 제대로 작동하는 일부 Gemmini 프로그램이 proxy-kernel에서 실패할 수 있습니다.

Building Gemmini Hardware and Cycle-Accurate Simulators
-----------------------------------------------

Verilator를 사용하여 사이클 정확도 Gemmini 시뮬레이터를 빌드하려면 아래 지침을 실행하십시오.

```shell
cd chipyard/generators/gemmini
./scripts/build-verilator.sh

# 또는 파형을 생성할 수 있는 시뮬레이터를 원한다면 다음을 실행하십시오:
# ./scripts/build-verilator.sh --debug
```

이 작업을 수행한 후, 사이클 정확도 시뮬레이터 외에도 SoC의 Verilog 설명을 `generated-src/`에서 찾을 수 있습니다.

Building Gemmini Functional Simulators
---------------------------

Gemmini의 기능적 ISA 시뮬레이터("Spike"라고 함)를 빌드하려면 아래 지침을 실행하십시오.

```shell
cd chipyard/generators/gemmini
./scripts/build-spike.sh
```

Spike는 Verilator나 VCS와 같은 사이클 정확도 시뮬레이터보다 _훨씬_ 빠르게 실행됩니다.
그러나 Spike는 기능적 정확성만 확인할 수 있으며, 정확한 성능 지표나 프로파일링 정보를 제공할 수 없습니다.

Run Simulators
---------------

이전에 빌드한 Gemmini RISCV 바이너리를 위에서 빌드한 시뮬레이터를 사용하여 실행하려면 아래 지침을 따르십시오:

```shell
cd chipyard/generators/gemmini

# 기능적 시뮬레이터에서 대규모 DNN 작업 부하 실행
./scripts/run-spike.sh resnet50

# 사이클 정확도 시뮬레이터에서 baremetal 모드로 작은 작업 부하 실행
./scripts/run-verilator.sh template

# proxy-kernel에서 작은 작업 부하를 사이클 정확도 시뮬레이터에서 실행
./scripts/run-verilator.sh --pk template

# 또는, `waveforms/`에 파형을 생성하려면:
# ./scripts/run-verilator.sh --pk --debug template
```

Next steps
--------

Gemmini를 사용하여 다양한 유형의 가속기를 구축하는 방법, Gemmini에 사용자 정의 데이터 유형을 추가하는 방법, Gemmini 프로그램을 작성하는 방법, Gemmini의 성능 카운터를 사용하여 작업 부하를 프로파일링하는 방법을 배우려면 [MLSys 2022 튜토리얼](https://sites.google.com/berkeley.edu/gemmini-tutorial-mlsys-2022) (또는 이전의 더 오래된 [IISWC 2021 튜토리얼](https://sites.google.com/berkeley.edu/gemminitutorialiiswc2021/))을 확인하십시오.

또는 [FireSim](fires.im)에 대해 알아보세요. 이 플랫폼은 FPGA 가속 사이클 정확도 시뮬레이션을 위한 플랫폼입니다.
우리는 Verilator/VCS에서 실행하는 데 너무 오랜 시간이 걸리는 종단 간 DNN 작업 부하를 실행하기 위해 FireSim을 사용합니다.
FireSim은 또한 Gemmini 하드웨어/소프트웨어가 Linux 환경에서 실행될 때 작동하는지 확인할 수 있습니다.

또는 Gemmini의 아키텍처, ISA 및 구성 매개변수에 대한 설명을 위해 이 문서의 나머지 부분을 계속 읽으십시오.

Architecture
================

Gemmini는 비표준 RISC-V 사용자 정의 명령어를 사용하는 RoCC 가속기로 구현됩니다.
Gemmini 유닛은 Rocket 또는 BOOM 타일의 RoCC 포트를 사용하며, 기본적으로 시스템 버스(즉, L2 캐시에 직접 연결)를 통해 메모리 시스템에 연결됩니다.

가속기의 핵심에는 행렬 곱셈을 수행하는 시스톨릭 배열이 있습니다.
기본적으로 행렬 곱셈은 _output-stationary_ 와 _weight-stationary_ 데이터 흐름을 모두 지원하며, 프로그래머는 런타임에 이들 간에 선택할 수 있습니다.
그러나 데이터 흐름은 설명 시점에 고정될 수도 있습니다.

시스톨릭 배열의 입력과 출력은 뱅킹된 SRAM으로 구성된 명시적으로 관리되는 스크래치패드에 저장됩니다.
DMA 엔진은 메인 메모리(호스트 CPU에서 볼 수 있는 메모리)와 스크래치패드 간의 데이터 전송을 용이하게 합니다.

weight-stationary 데이터 흐름은 시스톨릭 배열 외부에 누산기를 필요로 하기 때문에, 우리는 누산기 메모리 공간의 확장으로 간주될 수 있는 어드레싱이 가능한 최종 SRAM 뱅크를 추가합니다.
시스톨릭 배열은 누산기의 모든 주소에 결과를 저장할 수 있으며, 누산기에서 새로운 입력을 읽어올 수도 있습니다.
DMA 엔진은 또한 종종 바이어스를 로드하기 위해 필요한 누산기와 메인 메모리 간의 데이터를 직접 전송할 수 있습니다.

Gemmini는 또한 ReLU 또는 ReLU6과 같은 활성화 함수를 선택적으로 적용하거나, 양자화된 작업 부하를 지원하기 위해 결과를 2의 거듭제곱으로 축소하거나, 행렬을 전치하여 output-stationary 데이터 흐름을 지원하는 주변 회로를 포함합니다.

Generator Parameters
--------------------------

주요 관심 매개변수는 다음과 같습니다:

* 시스톨릭 배열 차원(``tileRows``, ``tileColumns``, ``meshRows``, ``meshColumns``): 시스톨릭 배열은 각 타일이 완전히 조합적인 2단계 계층 구조로 구성되며, 타일의 메시에는 각 타일 간에 파이프라인 레지스터가 있습니다.

![Gemmini의 시스톨릭 2단계 계층 구조](./img/gemmini-systolic-array.png)

* 데이터 흐름 매개변수(``dataflow``): Gemmini의 시스톨릭 배열이 output-stationary인지 weight-stationary인지를 결정하거나, 두 데이터 흐름을 모두 지원하여 프로그래머가 런타임에 선택할 수 있도록 합니다.

* 스크래치패드 및 누산기 메모리 매개변수(``sp_banks``, ``sp_capacity``, ``acc_capacity``): Gemmini 스크래치패드 메모리의 속성을 결정합니다: 스크래치패드 또는 누산기의 전체 용량(키비바이트 단위) 및 스크래치패드가 분할되는 뱅크 수를 포함합니다.

* 유형 매개변수(``inputType``, ``outputType``, ``accType``): Gemmini 가속기의 다양한 부분을 통과하는 데이터 유형을 결정합니다.
예를 들어, ``inputType``은 8비트 고정 소수점 숫자일 수 있으며, ``accType``은 행렬 곱셈에서 부분 합계를 결정하는 32비트 정수일 수 있습니다.
``outputType``은 처리 요소(PE) 간에 전달되는 데이터 유형만 결정합니다. 예를 들어, 8비트 곱셈은 16비트 결과를 생성하며, 이 결과는 시스톨릭 배열의 PE 간에 공유되어야 합니다.
    - 가능한 데이터 유형의 예는 다음과 같습니다:
        - `SInt(8.W)`: 부호 있는 8비트 정수
        - `UInt(32.W)`: 부호 없는 32비트 정수
        - `Float(8, 24)`: 단정밀도 IEEE 부동 소수점 수
    - 데이터 유형이 부동 소수점 숫자인 경우, PE 내에 추가할 시프트 레지스터 수를 지정하는 ``pe_latency`` 매개변수를 변경해야 할 수도 있습니다.
이것은 데이터 유형이 하나의 사이클 내에서 곱셈-누산 작업을 완료할 수 없는 경우에 필요할 수 있습니다.

* Access-execute 큐 매개변수(``ld_queue_length``, ``st_queue_length``, ``ex_queue_length``, ``rob_entries``): Access-execute 디커플링을 구현하기 위해 Gemmini 가속기는 로드 명령 큐, 스토어 명령 큐, 실행 명령 큐를 갖추고 있습니다. 이 큐의 상대적인 크기는 access-execute 디커플링의 수준을 결정합니다. Gemmini는 또한 리오더 버퍼(ROB)를 구현하며, ROB의 항목 수는 가능한 종속성 관리 제한을 결정합니다.

* DMA 매개변수(``dma_maxbytes``, ``dma_buswidth``, ``mem_pipeline``): Gemmini는 메인 메모리에서 Gemmini 스크래치패드로, Gemmini 누산기에서 메인 메모리로 데이터를 이동하기 위해 DMA를 구현합니다. 이 DMA 트랜잭션의 크기는 DMA 매개변수에 의해 결정됩니다. 이 DMA 매개변수는 Rocket Chip SoC 시스템 매개변수와 밀접하게 연결되어 있습니다. 특히 ``dma_buswidth``는 ``SystemBusKey`` ``beatBytes`` 매개변수와 관련이 있으며, ``dma_maxbytes``는 ``cacheblockbytes`` Rocket Chip 매개변수와 관련이 있습니다.

또한 설명 시점에 Gemmini에서 활성화하거나 제외할 수 있는 선택적 기능도 있습니다.
예를 들어:

* "이동 시" 스케일링(``mvin_scale_args``, ``mvin_scale_acc_args``):
DRAM 또는 메인 메모리에서 Gemmini의 로컬 스크래치패드 메모리로 데이터가 이동될 때 선택적으로 스케일링 인자를 곱할 수 있습니다.
이 매개변수는 스케일링 인자의 데이터 유형이 무엇인지, 스케일링이 실제로 어떻게 이루어지는지를 지정합니다.
이 값들이 ``None``으로 설정된 경우, 이 선택적 기능은 설명 시점에 비활성화됩니다.
스크래치패드 입력과 누산기 입력이 동일한 방식으로 스케일링되어야 하는 경우, ``mvin_scale_shared`` 매개변수를 ``true``로 설정하여 곱셈기와 기능 유닛이 공유되도록 할 수 있습니다.


Major Components
----------------

이 하위 섹션은 Gemmini의 RTL을 해킹하려는 사람들을 위한 것입니다.
여기에서는 Gemmini의 주요 하드웨어 구성 요소와 이들이 어떻게 결합되는지 간략히 설명합니다.
Gemmini의 하드웨어를 변경할 계획이 없다면(구성 매개변수를 변경하는 것 외에는), 이 섹션을 건너뛰어도 좋습니다.

### Decoupled Access/Execute

Gemmini는 디커플링된 액세스/실행 아키텍처로, "메모리 액세스"와 "실행" 명령이 하드웨어의 다른 영역에서 동시에 실행됩니다.
우리는 하드웨어를 크게 세 가지 "컨트롤러"로 나눕니다: "실행" 명령을 담당하는 컨트롤러, "로드" 명령을 담당하는 컨트롤러, 그리고 "스토어" 명령을 담당하는 컨트롤러입니다.
각 컨트롤러는 프로그래머로부터 직접 ISA 명령을 수신하고, 이를 디코딩하여 실행하며, 스크래치패드 및 누산기 SRAM에 대한 접근을 공유합니다.

* `ExecuteController`: 이 모듈은 행렬 곱셈과 같은 "실행" 유형의 ISA 명령을 실행하는 역할을 합니다.
이 모듈에는 도트 프로덕트(dots product)를 위한 시스톨릭 배열과 전치기(Transposer)가 포함됩니다.

* `LoadController`: 이 모듈은 메인 메모리에서 Gemmini의 개인 스크래치패드나 누산기로 데이터를 이동하는 모든 명령을 담당합니다.

* `StoreController`: 이 모듈은 Gemmini의 개인 SRAM에서 메인 메모리로 데이터를 이동하는 모든 명령을 담당합니다.
이 모듈은 또한 "맥스 풀링(max-pooling)" 명령을 담당합니다. 왜냐하면 Gemmini는 미포된 데이터를 개인 SRAM에서 메인 메모리로 이동할 때 풀링을 수행하기 때문입니다.

### Scratchpad and Accumulator

Gemmini는 시스톨릭 배열의 입력 및 출력을 저장하기 위해 "스크래치패드"와 "누산기"라고 불리는 일련의 개인 SRAM을 사용합니다.
일반적으로 입력은 스크래치패드에 저장되며, 부분 합계 및 최종 결과는 누산기에 저장됩니다.

스크래치패드와 누산기는 모두 `Scratchpad.scala`에 인스턴스화됩니다.
스크래치패드 뱅크는 `ScratchpadBank` 모듈에 의해 구현되며, 누산기 뱅크는 `AccumulatorMem` 모듈에 의해 구현됩니다.

스크래치패드와 누산기 SRAM의 각 행은 시스톨릭 배열의 너비를 따라 `DIM` "요소"로 구성됩니다.
각 "요소"는 Gemmini가 작동하는 단일 스칼라 값을 나타냅니다.

스크래치패드의 각 "요소"는 `inputType` 유형입니다(기본 설정에서는 8비트 정수).
누산기의 각 "요소"는 `accType` 유형입니다(기본 설정에서는 32비트 정수).

예를 들어, 기본 설정에서 16x16 시스톨릭 배열을 사용하는 경우, 스크래치패드 뱅크의 행 너비는 `16*bits(inputType)=128` 비트이고, 누산기 뱅크의 행 너비는 `16*bits(accType)=512` 비트입니다.

스크래치패드의 입력과 출력은 모두 `inputType`이어야 합니다.

누산기의 입력과 출력은 `accType` 또는 `inputType`일 수 있습니다.
`inputType` 값이 누산기로 입력되면 `accType`으로 변환됩니다.
`inputType` 값이 누산기에서 출력되면 먼저 `inputType` 값으로 "스케일링"됩니다.
기본 설정에서는 스케일링 함수가 단순히 `float32` 값을 곱하여 `int32`를 `int8`로 변환하는 함수로 구성됩니다.

스크래치패드 뱅크는 매우 간단하여 SRAM과 큐로만 구성됩니다.

누산기 뱅크는 좀 더 복잡합니다. 기본 SRAM 외에도, 이들은 인플레이스 누산을 지원하기 위한 일련의 더하기 장치들을 포함합니다.
또한 스케일러(앞서 설명한 대로)와 활성화 함수 유닛도 포함합니다.
스케일링 및 활성화 함수는 프로그래머가 누산기에서 데이터를 읽어올 때 `accType` 값을 `inputType` 값으로 변환하기를 원할 때 적용됩니다.
이는 일반적으로 한 레이어의 부분 합계 출력을 다음 레이어의 낮은 비트 폭의 양자화된 입력으로 변환하기 위해 수행됩니다.

### Systolic Array and Transposer

`MeshWithDelays`는 `ExecuteController` 내에서 인스턴스화되며, 시스톨릭 배열(`Mesh`), 전치기(`Transposer`), 그리고 시스톨릭 배열에 입력을 지연시키는 일련의 지연 레지스터로 구성됩니다.
`MeshWithDelays` 모듈은 한 번에 하나의 행씩 세 개의 행렬(`A`, `B`, 및 `D`)을 받아들이고, 한 번에 하나의 행씩 결과 `C = A * B + D`를 출력합니다.

weight-stationary 모드에서는 `B` 값이 시스톨릭 배열에 "사전 로드"되고, `A`와 `D` 값이 통과됩니다.
output-stationary 모드에서는 `D` 값이 시스톨릭 배열에 "사전 로드"되고, `A`와 `B` 값이 통과됩니다.

`A`, `B`, 및 `D`는 모두 `inputType` 유형이며, `C`는 `outputType` 유형입니다.
프로그래머가 `C`를 스크래치패드에 기록하려면, `C`는 `inputType`으로 변환됩니다.
그러나 프로그래머가 `C`를 누산기에 기록하려는 경우, `C`는 `accType`으로 변환됩니다.

weight-stationary 모드에서는 `inputType` D가 부분 합계를 정확하게 나타내기에 비트 폭이 부족한 경우가 많습니다.
따라서 weight-stationary 모드에서는 `D`가 보통 0 행렬이고, 누산기 SRAM이 대신 시스톨릭 배열의 부분 합계 출력을 누적하는 데 사용됩니다.

입력(`A`, `B`, 및 `D`)은 행렬의 각 입력이 정확한 시간에 올바른 PE에 도달하여 다른 행렬의 올바른 입력과 곱셈 및 누산되도록 지연 레지스터로 지연됩니다.
아래 다이어그램은 2x2 output-stationary 행렬 곱셈의 예(단, `D`는 제외)를 지연 레지스터와 함께 보여줍니다:

![지연 레지스터가 있는 시스톨릭 배열](./img/delay-registers.png)

시스톨릭 배열 자체는 `Mesh.scala`에 구현되어 있으며, 타일(`Tiles`)과 PE(`PEs`)의 2단계 계층 구조로 구성됩니다.
`Mesh`는 타일 세트로 구성되며, 각 타일 사이에 파이프라인 레지스터가 있습니다.
각 `Tile`은 조합 논리적 PE 세트로 구성되며, 각 PE는 weight-stationary 또는 output-stationary 데이터 흐름으로 단일 행렬 곱셈 작업을 수행합니다.

![시스톨릭 배열](./img/gemmini-systolic-array.png)

`MeshWithDelays` 모듈에는 카운터와 구성 레지스터도 포함됩니다.
`MeshWithDelays`는 각 행렬 곱셈 작업이 정확히 `DIM x DIM` 크기일 것으로 가정합니다. 여기서 `DIM`은 시스톨릭 배열의 너비입니다(기본 설정에서는 16).
이 카운터는 `DIM`까지 계산한 후 입력에서 `MeshWithDelays`로 구성 레지스터를 업데이트합니다.
이 구성 레지스터는 시스톨릭 배열에 입력하기 전에 `A`와 `B` 중 어느 것이 전치될지를 제어합니다.
또한, 시스톨릭 배열에 사전 로드된 값이 다음 행렬 곱셈을 위해 유지될지 아니면 덮어쓰고 교체될지를 제어합니다.

전치기 자체는 매우 간단한 시스톨릭 배열로 구현되며, 왼쪽에서 오른쪽으로 `DIM` 사이클 동안 입력을 전송한 후, 다시 위에서 아래로 `DIM` 사이클 동안 전송합니다.
이것은 아래 다이어그램에 나와 있습니다:

![전치기](./img/transposer.png)

output-stationary 행렬 곱셈의 경우, 프로그래머가 전치를 요청하지 않더라도 전치기가 사용된다는 점에 유의하십시오.
이는 시스톨릭 배열이 output-stationary 모드에서 `A`의 동일한 행의 입력을 동일한 PE에 입력하는 것을 예상하기 때문입니다. 그러나 `A`의 단일 행의 모든 값이 동일한 스크래치패드 SRAM 행에 저장됩니다.
따라서 행렬 요소를 동일한 행에 있는 요소들이 연속적으로 PE에 입력되도록 스크래치패드에서 읽은 후 전치해야 합니다.

### DMA

Gemmini에는 메인 메모리에서 Gemmini의 개인 SRAM으로 데이터를 읽어오는 DMA와, Gemmini의 개인 SRAM에서 메인 메모리로 데이터를 이동하는 DMA가 포함되어 있습니다.
이 모듈들은 모두 `DMA.scala`에 구현되어 있습니다.

두 DMA는 가상 주소에서 작동하며, 이를 물리적 메인 메모리 주소로 변환하기 위해 TLB에 접근합니다.
TLB에서 미스가 발생하면, Gemmini의 호스트 CPU와 공유되는 PTW로 투명하게 대체됩니다.

Gemmini의 개인 TLB에서 물리적 주소를 얻은 후, DMA는 큰 메모리 요청을 더 작은 [TileLink](https://sifive.cdn.prismic.io/sifive%2Fcab05224-2df1-4af8-adee-8d9cba3378cd_tilelink-spec-1.8.0.pdf) 읽기 및 쓰기 요청으로 나눕니다.
TileLink 프로토콜을 만족시키기 위해, 각 메모리 요청은 메인 메모리에서 요청된/저장된 바이트 수로 정렬되어야 하며, 각 메모리 요청의 크기(바이트 단위)는 2의 거듭제곱이어야 합니다.
DMA는 일반적으로 가능하면 TileLink 요청의 수를 최소화하려고 합니다. 이는 메인 메모리에서 더 많은 데이터를 읽어야 하더라도 성능을 저하시킬 수 있는 TileLink 요청의 과도한 수를 방지하기 위함입니다.

DMAWriter, 즉 개인 SRAM에서 메인 메모리로 데이터를 쓰는 역할을 하는 모듈에는 메모리 쓰기 작업 중에 데이터에 대한 맥스 풀링을 수행하기 위한 `>` 비교기가 포함되어 있습니다.

### ROB

Gemmini의 디커플링된 액세스-실행 아키텍처로 인해, `LoadController`, `StoreController`, `ExecuteController`의 명령이 다른 컨트롤러의 명령과 비교하여 동시에 실행되고 순서 없이 처리될 수 있습니다.
Gemmini는 다른 컨트롤러의 명령 사이의 종속성을 감지하기 위한 리오더 버퍼(ROB)를 포함합니다.
ROB에 있는 명령은 다른 컨트롤러의 명령에 종속성이 없을 때에만 해당 컨트롤러에 발행됩니다.

동일한 컨트롤러에 대상이 된 명령은 순서대로 발행됩니다.
ROB는 동일한 컨트롤러 내에서 명령 간의 종속성을 검사하지 않습니다. 각 컨트롤러는 프로그램 순서대로 명령을 수신한다는 가정 하에 자체 내부 종속성과 위험을 처리해야 하기 때문입니다.

### Matmul and Conv Loop Unrollers

Gemmini의 시스톨릭 배열은 최대 `DIMxDIM` 요소의 행렬 곱셈만 수행할 수 있습니다.
이보다 더 큰 행렬 곱셈과 컨볼루션을 수행할 때는 프로그래머가 자신의 행렬 곱셈을 더 작은 `DIMxDIM` 행렬 곱셈 시퀀스로 타일링해야 합니다.

그러나 이러한 연산을 효율적으로 타일링하는 것은 CPU 및 루프 오버헤드와 소프트웨어 루프를 풀어내고 파이프라이닝하는 어려움으로 인해 프로그래머에게 어려울 수 있습니다.

이 어려움을 줄이기 위해, Gemmini의 ISA에는 대규모 행렬 곱셈과 컨볼루션을 자동으로 타일링하고 풀어내는 고수준 CISC 유형의 명령이 포함되어 있습니다.
이들은 `LoopMatmul` 및 `LoopConv` 모듈에 의해 구현됩니다.

이 모듈들은 FSM으로 구현되었으며, 성능을 극대화하기 위해 행렬 곱셈/컨볼루션 타일을 더블 버퍼링하고, 메모리 액세스와 도트 프로덕트 계산 간의 최대 중첩을 위해 ROB의 로드/스토어/실행 명령의 비율을 모니터링합니다.
예를 들어, ROB이 행렬 곱셈 명령으로 가득 차 있어 들어오는 로드 명령에 슬롯을 남기지 않는 경우, FSM은 Gemmini의 데이터 경로에 있는 동시 로드 명령을 위해 더 많은 공간을 허용하기 위해 행렬 곱셈 명령의 발행을 일시 중지할 것입니다.

Software
==========

Gemmini ISA는 아래 `ISA` 섹션에서 설명됩니다.
이 ISA에는 구성 명령, 데이터 이동 명령(메인 메모리에서 Gemmini의 개인 메모리로 이동하거나 그 반대 방향), 및 행렬 곱셈 실행 명령이 포함됩니다.

Gemmini 명령어는 GNU binutils 어셈블러를 통해 노출되지 않으므로, 이러한 명령어 인코딩을 구성하기 위해 여러 C 매크로가 제공됩니다.

Gemmini 생성기는 이러한 사용자 정의 Gemmini 명령 호출을 공통 DNN 연산자(예: 행렬 곱셈, 컨볼루션(풀링 포함 또는 제외), 행렬 덧셈 등)로 감싸는 C 라이브러리를 포함합니다.
생성기의 ``software`` 디렉토리에는 위에서 언급한 라이브러리 및 매크로, baremetal 테스트, 그리고 Linux 환경에서 테스트를 실행하기 위한 일부 FireMarshal 워크로드가 포함되어 있습니다. 특히 C 라이브러리는 ``software/gemmini-rocc-tests/include/gemmini.h`` 파일에 있습니다.

Gemmini 생성기는 생성기 매개변수를 기반으로 C 헤더 파일을 생성합니다. 이 헤더 파일은 C 라이브러리와 함께 컴파일되어 라이브러리 성능을 조정합니다. 생성된 헤더 파일은 ``software/gemmini-rocc-tests/include/gemmini_params.h``에서 찾을 수 있습니다.

Gemmini는 또한 Microsoft의 ONNX-Runtime 프레임워크의 포트를 통해 ONNX에서 지정된 신경망을 실행하는 데 사용할 수 있습니다. 이 포트는 `software` 디렉토리에 서브모듈로 포함된 [onnxruntime-riscv](https://github.com/pranav-prakash/onnxruntime-riscv) 리포지토리에 포함되어 있습니다.
ONNX-Runtime 사용을 시작하려면 `git submodule update --init --recursive software/onnxruntime-riscv` 명령을 실행하고 [여기](https://github.com/pranav-prakash/onnxruntime-riscv/blob/systolic/systolic_runner/docs)에서 문서를 참조하십시오.

## Build and Run Gemmini Tests

Gemmini 테스트를 빌드하려면:

```shell
cd software/gemmini-rocc-tests/
./build.sh
```

이후, 테스트 바이너리는 `software/gemmini-rocc-tests/build` 디렉토리에 생성됩니다.
이름이 `-baremetal`로 끝나는 바이너리는 bare-metal 환경에서 실행되도록 설계되었으며, 이름이 `-linux`로 끝나는 바이너리는 Linux 환경에서 실행되도록 설계되었습니다.
테스트는 사이클 정확도의 RTL 시뮬레이터 또는 (훨씬 더 빠른) 기능적 ISA 시뮬레이터인 Spike에서 실행할 수 있습니다.

우리는 Gemmini 명령어를 지원하는 특별한 Spike 확장을 사용합니다. [여기](https://github.com/ucb-bar/libgemmini)에서 찾을 수 있습니다. Chipyard를 사용하는 경우, Chipyard의 루트 디렉토리에서 `./scripts/build-toolchains.sh riscv-tools`를 실행한 다음, Gemmini 디렉토리에서 `make -C software/libgemmini install`을 실행하여 Spike를 쉽게 빌드할 수 있습니다.
그런 다음, Gemmini의 스크래치패드로 행렬을 이동시키고, 다시 메인 메모리로 이동시키는 단순한 테스트인 `mvin_mvout` 테스트를 실행하려면 다음 명령을 실행하십시오:

```shell
cd build/bareMetalC
spike --extension=gemmini mvin_mvout-baremetal
```

## Writing Your Own Gemmini Tests

`software/gemmini-rocc-tests/bareMetalC/template.c`는 Gemmini 테스트를 작성할 때 기반으로 사용할 수 있는 템플릿입니다. 자신의 Gemmini 테스트를 작성하려면 다음을 실행하십시오:

```shell
cd software/gemmini-rocc-tests/
cp bareMetalC/template.c bareMetalC/my_test.c
```

그런 다음, `bareMetalC/Makefile`의 상단에 있는 `tests` 목록에 `my_test`를 추가하십시오. 이후, `./build.sh`를 실행하면 `build/bareMetalC` 디렉토리에 `my_test-baremetal`이 설치됩니다.

## DNN Tests

ResNet50과 같은 예제 DNN은 `software/gemmini-rocc-tests/imagenet` 및 `software/gemmini-rocc-tests/mlps`에 있습니다.
이 테스트들은 위에서 설명한 다른 테스트들과 동일한 방식으로 빌드되고 실행되지만, VCS나 Verilator와 같은 소프트웨어 시뮬레이터에서는 실행 시간이 너무 오래 걸립니다.
이 테스트들은 대신 [Firesim](https://fires.im/)이라는 FPGA 가속 시뮬레이션 플랫폼을 통해 실행할 것을 권장합니다. 이 플랫폼은 실행 시간을 며칠에서 몇 분으로 단축시켜줍니다.

DNN 테스트는 `gemmini.h`에 있는 공통 DNN 연산자들의 C 라이브러리를 기반으로 합니다.
이들은 거의 직접적인 Gemmini ISA 명령어 호출을 하지 않으며, 대부분 C 라이브러리에 있는 래퍼를 호출합니다.

# Memory Addressing Scheme

Gemmini의 개인 메모리는 "행 단위 주소 지정" 방식으로, 각 행은 시스톨릭 배열의 너비인 `DIM` 요소로 구성됩니다. 기본 설정에서는 `DIM`이 16입니다.
스크래치패드에서는 이러한 요소들이 `inputType` 유형이며, 누산기에서는 `accType` 유형입니다.

Gemmini의 모든 개인 메모리 주소는 32비트 길이를 가지며, 가장 상위 3비트는 예약되어 있으며 특별한 의미를 가집니다:
* 비트 31(MSB)은 스크래치패드를 주소 지정하는 경우 0, 누산기를 주소 지정하는 경우 1입니다.
* 비트 30은 스크래치패드를 주소 지정하는 경우 또는 누산기에서 읽어오는 경우 무시됩니다. 반대로 누산기에 쓰는 경우, 비트 30이 0이면 해당 주소의 데이터를 덮어쓰고, 1이면 해당 주소의 데이터에 누적합니다.
* 비트 29는 스크래치패드를 주소 지정하는 경우 또는 누산기에 쓰는 경우 무시됩니다. 반대로 누산기에서 읽어오는 경우, 비트 29가 0이면 누산기에서 스케일 다운된 `inputType` 데이터를 읽어오고, 1이면 `accType` 데이터를 읽어옵니다.
    - 비트 29가 1인 경우, 누산기의 출력에 활성화 함수나 스케일링이 적용되지 않습니다.

다음 그림은 2x2 시스톨릭 배열을 사용하는 Gemmini 설정의 메모리 주소 지정 방식을 보여줍니다:

![Gemmini의 메모리 주소 지정 방식](./img/memory-addressing.png)

Gemmini는 메인 메모리 주소(이는 CPU에서도 볼 수 있는 주소)에 소프트웨어에서 볼 수 있는 가상 주소를 통해 접근합니다.
물리 주소 변환은 Gemmini에서 투명하게 처리됩니다.

# ISA

이 섹션에서는 Gemmini의 어셈블리 레벨 ISA, 즉 RISC-V 사용자 정의 명령어로 구성된 ISA에 대해 설명합니다.

## Data Movement
### `mvin` 메인 메모리에서 스크래치패드로 데이터 이동
**형식:** `mvin rs1, rs2`
- `rs1` = 스크래치패드로 로드할 가상 DRAM 주소(바이트 주소)
- `rs2[31:0]` = 로컬 스크래치패드 또는 누산기 주소
- `rs2[47:32]` = 로드할 열의 수
- `rs2[63:48]` = 로드할 행의 수. `DIM`보다 작거나 같아야 합니다.
- `funct` = 2

**동작:** Scratchpad[rs2] <= DRAM[Translate[rs1]]
- 메인 메모리에서 Gemmini의 개인 메모리로 2D 행렬을 로드합니다.
- 로드는 rs1/rs2 기본 주소에서 순차적으로 이루어집니다.
- 메인 메모리 스트라이드는 `config_mvin` 명령어로 설정해야 합니다.
- 로드할 열의 수가 `DIM`보다 크면 여러 서브매트릭스가 이동됩니다.
이 서브매트릭스 간의 개인 메모리 스트라이드는 `config_mvin` 명령어로 설정됩니다.

다음 그림은 `mvin` 명령어의 동작 방식을 보여줍니다:

![Gemmini의 mvin 명령어](./img/mvin.png)

추가로, 다음 그림은 로드할 열의 수가 `DIM`보다 클 때의 특수 사례를 보여줍니다:

![여러 열을 가진 Gemmini의 mvin 명령어](./img/block-mvin.png)

**참고:**
* Gemmini에는 실제로 **세 가지** `mvin` 명령어가 있습니다: `mvin`, `mvin2`, 및 `mvin3`.
`mvin2`와 `mvin3`는 `mvin`과 완전히 동일하지만, 각각 독립적인 설정 레지스터 세트를 가지고 있습니다.
`config_mvin`(아래에 설명됨)을 호출할 때, 프로그래머는 설정하려는 `mvin` 명령어를 선택할 수 있습니다.
* 세 가지 `mvin` 명령어가 있는 이유는 프로그래머가 A, B, 및 D 행렬의 로드를 중첩할 수 있도록 하기 위함입니다(예를 들어, `A*B+D` 행렬 곱셈의 경우, A, B, D는 각각 다른 메인 메모리 스트라이드를 가질 수 있습니다).

### `mvout` 스크래치패드에서 L2/DRAM으로 데이터 이동
**형식:** `mvout rs1, rs2`
- `rs1` = 스크래치패드에서 메인 메모리로 쓰기 위한 가상 DRAM 주소(바이트 주소)
- `rs2[31:0]` = 로컬 스크래치패드 주소
- `rs2[47:32]` = 저장할 열의 수
- `rs2[63:48]` = 저장할 행의 수
- `funct` = 3

**동작:** DRAM[Translate[rs1]] <= Scratchpad[rs2]
- 스크래치패드에서 메인 메모리로 2D 행렬을 저장합니다.
- 저장은 rs1/rs2 기본 주소에서 순차적으로 이루어집니다. 스트라이드는 `config_mvout` 명령어로 설정해야 합니다.

## Configuration
### `config_ex` 실행 파이프라인 구성
**형식:** `config_ex rs1 rs2`
- `rs1[1:0]`은 `00`이어야 합니다.
- `rs1[2]`는 출력(0) 또는 가중치(1) 고정 여부를 결정합니다.
- `rs1[3]` = 활성화 함수: relu(1) 또는 활성화 함수 없음(0)
- `rs1[8]` = A를 전치할지 여부
- `rs1[9]` = B를 전치할지 여부
- `rs1[31:16]` = 시스톨릭 배열에 행렬 A의 행을 공급하는 동안 사용하는 스크래치패드의 행 간 간격.
여기서 "A"는 행렬 곱셈 A * B = C에서 왼쪽 행렬 A를 나타냅니다.
이 간격이 1이면, 스크래치패드에서 연속적인 행이 시스톨릭 배열에 공급됩니다.
간격이 2이면, 시스톨릭 배열에 공급되는 행은 하나씩 건너뜁니다.
- `rs1[63:32]` = 누산기의 `accType` 출력을 `inputType` 값으로 축소하는 데 사용하는 스칼라 값.
    - 기본 설정에서는 `rs1[63:32]`가 `float32` 유형입니다.
- `rs2[31:0]` = 시스톨릭 배열을 떠날 때 행렬 곱셈 결과를 우측으로 시프트할 비트 수
    - 이 매개변수는 출력 고정 모드에서만 관련이 있으며, 시스톨릭 배열 내에서 부분 합계를 누적하고 스크래치패드에 기록할 때 축소해야 합니다.
- `funct` = 0

**동작:** 모드 <= rs1(2); 시프트 <= rs2; A_stride <= rs1[31:16]

**참고사항:**
- 현재로서는 특정 전치 옵션의 조합을 올바른 데이터 흐름을 선택하지 않으면 수행할 수 없습니다.
이 제한은 나중에 해제될 수 있습니다.

| 데이터 흐름 | 전치 A | 전치 B | 허용 여부 |
| :---: | :---: | :---: | :---: | 
| OS | 아니오 | 아니오 | 예 |
| OS | 아니오 | 예 | 아니오 |
| OS | 예 | 아니오 | 예 |
| OS | 예 | 예 | 예 |
| WS | 아니오 | 아니오 | 예 |
| WS | 아니오 | 예 | 예 |
| WS | 예 | 아니오 | 예 |
| WS | 예 | 예 | 아니오 |

### `config_mvin` 로드 파이프라인 구성
**형식:** `config_mvin rs1 rs2`
- `rs1[1:0]`은 `01`이어야 합니다.
- `rs1[2]`는 `mvin`이 누산기 유형 `accType`인지, 입력 유형 `inputType`인지 결정합니다.
- `rs1[4:3]`은 0이면 `mvin`의 스트라이드 설정, 1이면 `mvin2`의 스트라이드 설정, 2이면 `mvin3`의 스트라이드 설정을 의미합니다.
- `rs1[31:16]`은 스크래치패드 메모리 스트라이드(위에서 말한 "개인 메모리 스트라이드")
- `rs1[63:32]`은 스크래치패드로 이동할 때 데이터에 곱할 스케일입니다. Gemmini가 `mvin` 동안 값을 스케일링할 수 있도록 구성되지 않은 경우 이 값은 무시됩니다.
- `rs2`는 메인 메모리 스트라이드(바이트 단위)
- `funct` = 0

**동작:** 스트라이드 <= rs2; 스케일 <= rs1[63:32]

### `config_mvout` 스토어 파이프라인 구성
**형식:** `config_mvout rs1 rs2`
- `rs1[1:0]`은 `10`이어야 합니다.
- `rs2` = 스트라이드(바이트 단위)
- `funct` = 0

`mvout` 작업 동안, Gemmini는 또한 맥스 풀링을 수행할 수 있습니다.
**이 기능은 실험적이며 변경될 수 있습니다.**
이 기능은 데이터가 NHWC 형식으로 스크래치패드 또는 누산기에 저장된다고 가정합니다.
이 기능을 제어하는 매개변수는 다음과 같습니다:

- `rs1[5:4]` = 맥스 풀링 스트라이드. 0인 경우 맥스 풀링이 비활성화됩니다.
- `rs1[7:6]` = 맥스 풀링 윈도우 크기
- `rs1[9:8]` = 상단의 제로 패딩
- `rs1[11:10]` = 좌측의 제로 패딩
- `rs1[31:24]` = 풀링 후 이미지의 출력 차원
- `rs1[39:32]` = 출력할 풀링된 행의 수
- `rs1[47:40]` = 출력할 풀링된 열의 수
- `rs1[55:48]` = 풀링할 비풀링된 행의 수
- `rs1[63:56]` = 풀링할 비풀링된 열의 수

**동작:** 스트라이드 <= rs2; 맥스 풀링 매개변수 <= rs1

### `config_norm` 정규화 명령 구성
**형식:** `config_norm rs1 rs2`

`config_norm`은 **실험적** 명령으로, 주로 [I-BERT](https://arxiv.org/abs/2101.01321)라는 정수 전용 BERT 변형을 Gemmini에서 지원하기 위해 추가되었습니다.
이 명령은 I-BERT의 GELU, 레이어 정규화, 소프트맥스 변형에 사용되는 스칼라 상수를 설정할 수 있습니다.

### `flush` TLB 플러시
**형식:** `flush rs1`
- `rs1` = `rs1[0]`이 1인 경우, 현재 TLB 요청이 스킵됩니다(페이지 폴트가 발생하여 인터럽트를 기다리고 있는 경우). 그렇지 않은 경우, 현재 TLB 요청이 반복됩니다.

**참고:**

- 이 명령은 다른 큐에 대기 중인 명령 없이 _즉시_ 실행됩니다.
프로그래머가 필요한 경우 펜스를 삽입할 책임이 있습니다.

## Core Matmul Sequences
모든 행렬 곱셈 연산은 `matmul.preload`와 `matmul.compute`의 조합으로 구성됩니다(단일 명령어의 길이가 길기 때문에 두 개의 명령어로 나뉘었습니다).
`matmul.preload`는 `matmul.compute`보다 먼저 실행되어야 합니다.

예제:
```
//// OS 행렬 곱셈 예제 ////
// rs1 = InputD
// rs2 = OutputC
// rs3 = InputA
// rs4 = InputB
// 행렬 곱셈 InputA * InputB + InputD = OutputC
1. matmul.preload $rs1 $rs2
2. matmul.compute $rs3 $rs4
```
**동작:** Scratchpad[rs2] <= Scratchpad[rs3] \* Scratchpad[rs4] + Scratchpad[rs1]

**주소 지정에 대한 참고사항:**
- B 또는 D의 경우, 주소를 모두 높은 비트로 대체하여 0 행렬을 입력할 수 있습니다.
- A의 경우, 주소를 모두 높은 비트로 대체하여 정의되지 않은 임의의 데이터를 입력할 수 있습니다.

### Preloading
**형식:** `matmul.preload rs1, rs2`
- `rs1[31:0]` = D 행렬의 로컬 스크래치패드 주소(output-stationary의 경우) 또는 B 행렬(weight-stationary의 경우)
- `rs1[47:32]` = D/B 행렬의 열 수
- `rs1[63:48]` = D/B 행렬의 행 수
- `rs2[31:0]` = C 행렬의 로컬 스크래치패드 주소.
이 값이 모두 높은 비트로 설정되면 C가 스크래치패드나 누산기에 기록되지 않습니다.
- `rs2[47:32]` = C 행렬의 열 수
- `rs2[63:48]` = C 행렬의 행 수
- `funct` = 6

**커밋 동작:** 이 명령은 시스톨릭 배열이 명령을 수신한 후 다음 사이클에 커밋됩니다. 시스톨릭 배열은 이후 OS/WS 특정 명령이 도착할 때까지 유휴 상태로 유지됩니다.

### Computing
#### 명시적 사전 로딩
**형식:** `matmul.compute.preloaded rs1, rs2`
- `rs1[31:0]` = A 행렬의 로컬 스크래치패드 주소(시스톨릭 배열 단일 축 주소 지정)
- `rs1[47:32]` = A 행렬의 열 수
- `rs1[63:48]` = A 행렬의 행 수
- `rs2[31:0]` = B 행렬의 로컬 스크래치패드 주소(시스톨릭 배열 단일 축 주소 지정) 또는 D 행렬(weight-stationary의 경우)
- `rs2[47:32]` = B/D 행렬의 열 수
- `rs2[63:48]` = B/D 행렬의 행 수
- `funct` = 4
- 이 명령은 사전 로딩된 값(D 또는 B)으로 연산을 수행합니다.

#### 이전 사전 로딩 재사용

**형식:** `matmul.compute.accumulated rs1, rs2`
- `funct` = 5
- `rs1`과 `rs2`는 `matmul.compute.preloaded`와 동일한 인코딩을 가집니다.
- output-stationary의 경우, 이 명령은 시스톨릭 배열의 이전에 계산된 결과(C)로 연산을 수행하며, 그 위에 누적합니다.
- weight-stationary의 경우, 이 명령은 시스톨릭 배열의 이전에 사전 로딩된 가중치(B)로 연산을 수행합니다.

## Loop Instructions

Gemmini에는 `DIMxDIM`보다 훨씬 큰 데이터를 대상으로 행렬 곱셈 및 컨볼루션을 수행할 수 있는 CISC 유형의 명령이 포함되어 있습니다.

이러한 CISC 명령은 프로그래머가 위에서 설명한 다른 ISA 명령을 통해 타일링하고 루프를 구성할 수 있는 모든 작업을 수행할 수 있지만, 비전문 프로그래머가 작성한 타일링된 루프보다 더 높은 처리량을 달성할 수 있습니다.
CISC 명령은 성능 향상을 위한 도구로 간주해야 하며, 가속기에 추가적인 기능을 제공하지 않습니다.

CISC 명령에는 하나의 RISC-V 사용자 정의 명령어로는 너무 많은 피연산자가 있습니다.
따라서 이들은 일련의 많은 RISC-V 사용자 정의 명령어로 구현되며, 프로그래머가 연속적으로 호출해야 합니다.

이 명령들은 `software/gemmini-rocc-tests/include/gemmini.h`에서 찾을 수 있으며, 예제 사용법도 포함되어 있습니다.
아래에 그들의 인수를 나열합니다.

**이 루프 명령어들은 실험적이며 변경될 수 있습니다.**

### `gemmini_loop_ws` 행렬 곱셈 루프(WS 데이터 흐름)

이 명령은 `A * B + D = C`를 계산하지만, `A`, `B`, `D`, `C`는 모두 `DIMxDIM`보다 클 수 있습니다.
`A`와 `B`는 `inputType`이어야 하지만, `D`와 `C`는 `inputType` 또는 `accType`일 수 있습니다.

이 행렬들의 크기는 `I`, `J`, `K`로 표현됩니다:

```
A의 스크래치패드 행 수 = I * K * DIM
B의 스크래치패드 행 수 = K * J * DIM
D의 누산기 행 수 = I * J * DIM
C의 누산기 행 수 = I * J * DIM
```

그러나 단일 `gemmini_loop_ws`로 처리할 수 있는 총 스크래치패드 행 수는 전체 스크래치패드 크기의 **절반**을 초과할 수 없습니다. 이는 Gemmini가 CISC 명령어 동안 더블 버퍼링을 수행하기 때문입니다.
더 큰 행렬 곱셈을 수행하려면, 루프 명령어를 외부 루프 내에서 타일링해야 합니다.

`gemmini_loop_ws` 명령의 외부 타일링을 지원하기 위해 `ex_accumulate`라는 인수가 포함되어 있으며, 이는 동일한 외부 루프 내에서 `gemmini_loop_ws`를 여러 번 호출하여 이미 누산기에 있는 부분 합계 위에 행렬 곱셈을 수행할지 여부를 결정합니다.

### `gemmini_loop_conv_ws` 컨볼루션 루프(WS 데이터 흐름)

Gemmini는 컨볼루션을 위한 CISC 명

령도 포함하고 있으며, 이는 행렬 곱셈 CISC 명령과 유사하게 구현되었습니다.
`gemmini_loop_conv_ws`는 WS 데이터 흐름으로 컨볼루션을 수행하며, 맥스 풀링, 전치 컨볼루션 및 가중치와 입력 데이터에 대한 다양한 전처리 변환과 같은 기능도 지원합니다.

`gemmini_loop_ws`와 마찬가지로, 단일 `gemmini_loop_conv_ws` 호출의 입력은 Gemmini의 개인 메모리의 절반에 맞아야 하며, 이는 더블 버퍼링을 지원하기 위해서입니다.
프로그래머가 더 큰 컨볼루션을 수행하려는 경우, 외부 루프 내에서 `gemmini_loop_conv_ws`를 타일링하고 래핑해야 합니다.


# Citing Gemmini

Gemmini가 귀하의 학술 연구에 도움이 되었다면, 논문에서 Gemmini를 인용하는 것을 권장합니다. 다음은 인용 예제입니다:

```
@INPROCEEDINGS{gemmini-dac,
  author={Genc, Hasan and Kim, Seah and Amid, Alon and Haj-Ali, Ameer and Iyer, Vighnesh and Prakash, Pranav and Zhao, Jerry and Grubb, Daniel and Liew, Harrison and Mao, Howard and Ou, Albert and Schmidt, Colin and Steffl, Samuel and Wright, John and Stoica, Ion and Ragan-Kelley, Jonathan and Asanovic, Krste and Nikolic, Borivoje and Shao, Yakun Sophia},
  booktitle={Proceedings of the 58th Annual Design Automation Conference (DAC)}, 
  title={Gemmini: Enabling Systematic Deep-Learning Architecture Evaluation via Full-Stack Integration}, 
  year={2021},
  volume={},
  number={},
  pages={}
}
```

# Acknowledgements

- 이 프로젝트는 부분적으로 DARPA RTML 프로그램(계약 번호 FA8650-20-2-7006) 하에 미 정부의 자금 지원을 받았습니다. 이 문서에 포함된 견해와 결론은 저자의 것이며, 미 정부의 공식 정책을 나타내는 것으로 해석되어서는 안 됩니다.
- Gemmini [로고](./img/full-logo.svg)는 Dima Nikiforov([@CobbledSteel](https://github.com/CobbledSteel))가 디자인했습니다.


