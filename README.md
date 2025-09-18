treestream
=======
A simple interface to Root files containing simple trees, such as the
CMS NanoAOD or the Root files created using __Delphes__. The enviroment variable __TREESTREAM_PREFIX__ should be set to the directory in which you wish to install treestream, that is, to the directory containing the bin, lib, and include directories. If you do not use an environment management system such as miniconda3 (a slim version of Anaconda), we suggest
that you create a directory called __external__ in your home directory,
as shown below to contain all external packages, and install
treestream in that directory.  You should clone (download) external packages to __external__, but do not try to install treestream within the treestream directory itself!

INSTALLATION
```bash
	cd
	mkdir -p external/bin
	mkdir -p external/lib
	mkdir -p external/include
	mkdir -p external/share

	cd $HOME/external
	git https://github.com/hbprosper/treestream.git
	cd treastream
	export TREESTREAM_PREFIX=$HOME/external (or $CONDA_PREFIX if you use miniconda3)
	make
	make install
```
TEST
```bash
	cd test
	./testtreestream
	./testdelphes
	./testvector
```
There is also a __jupyter__ notebook version of the test program.

ANALYZER UTILITIES

1. __mkvariables.py__  reads a Root file and creates the file __variables.txt__
containing a description of (by default) the first tree it finds.

2. __mkanalyzer.py__ reads __variables.txt__ and creates the skeleton of an C++
and Python analyzer program for the Root tree.

## 코드 구조 쉽게 이해하기 (Korean Guide)

### 전체 흐름 한눈에 보기
이 저장소는 CMS 실험에서 자주 사용하는 NanoAOD, miniAOD와 같은 다양한 형태의 ntuple을 빠르게 분석할 수 있도록 *뼈대 코드(skeleton)*를 생성하고, 공통으로 재사용할 수 있는 보조 함수들을 제공합니다.

1. **`bin/mkanalyzer.py`** – 분석기(Analyzer) 뼈대 생성기
   - `variables.txt`를 읽고, C++/Python으로 된 예제 분석기를 만들어 줍니다.
   - 실행 옵션을 다루는 코드, 이벤트 루프, 출력 파일 관리, 그리고 선택한 브랜치(branch)만 불러오는 기능까지 기본 구조를 모두 갖춘 템플릿을 만들어 줍니다.
   - `--debug` 플래그가 켜지면 처리 속도나 주요 단계에 대한 로그를 더 자세히 출력하도록 설계되어 있습니다.

2. **`tnm/tnm.h` & `tnm/tnm.cc`** – 분석기에서 공용으로 사용하는 유틸리티 모음
   - 출력 파일 관리(`outputFile`), 명령줄 파싱(`commandLine`), 사전적인 물리 계산 함수(`deltaR`, `setStyle` 등)를 제공합니다.
   - `commandLine` 구조체는 `--debug` 옵션을 인식하여 분석기가 디버그 로그를 활성화할 수 있게 합니다.

3. **기타 디렉터리**
   - `src`, `include`: ROOT 기반 스트리밍 기능에 필요한 핵심 소스와 헤더가 들어 있습니다.
   - `test`: 제공되는 테스트 프로그램을 통해 treestream이 올바르게 설치되었는지 확인할 수 있습니다.

### `eventBuffer`와 브랜치 선택 이해하기
분석기가 만들어 내는 C++ 코드의 중심에는 `eventBuffer` 클래스가 있습니다. 이 클래스는 다음과 같은 일을 합니다.

- **브랜치 연결(SetBranchAddress)**: ROOT 트리에서 사용할 변수(브랜치)를 미리 연결해 두고, 이벤트를 읽어 들일 때 값이 자동으로 채워지도록 합니다.
- **선택한 브랜치만 불러오기**: 사용자가 `--branches muon`처럼 특정 접두어(prefix)를 주면, `muon`으로 시작하는 브랜치만 활성화해서 메모리를 절약합니다. 아무 접두어도 주지 않으면 모든 브랜치를 불러옵니다.
- **인덱스 맵(index map)**: 예를 들어 여러 개의 제트(또는 전자)가 있는 이벤트에서, 선택한 객체의 인덱스를 추적하여 후속 계산에서 쉽게 참조할 수 있습니다.

### `std::unordered_map` 초보자용 설명
브랜치 선택을 효율적으로 관리하기 위해 코드에는 `std::unordered_map`이 사용됩니다. 간단히 말해, **특정한 이름(key)**과 **그에 대응하는 값(value)**을 빠르게 찾을 수 있게 도와주는 자료구조입니다.

- **왜 사용할까?** 예를 들어 브랜치 이름을 키로, "이 브랜치를 읽을지 말지"라는 불리언 값을 저장해 둔다고 합시다. `unordered_map`은 원하는 브랜치 이름을 거의 즉시 찾아볼 수 있어, "이 브랜치를 켜야 할까?"를 빠르게 판단할 수 있습니다.
- **어떻게 생겼을까?** 파이썬의 `dict`와 비슷하게 키-값 쌍을 저장합니다. 단, C++에서는 각 키와 값의 자료형을 명확히 적어줘야 합니다. 이 프로젝트에서는 대략 `std::unordered_map<std::string, bool>`과 같은 형태로, 문자열 키(브랜치 이름)와 불리언 값(사용 여부)을 저장합니다.
- **사용 예시**
  ```cpp
  std::unordered_map<std::string, bool> branchEnabled;
  branchEnabled["Muon_pt"] = true;   // "Muon_pt" 브랜치를 읽겠다는 뜻
  branchEnabled["Photon_pt"] = false; // "Photon_pt" 브랜치는 건너뛰겠다는 뜻

  if (branchEnabled["Muon_pt"]) {
      // 실제로 브랜치를 ROOT 트리에 연결하는 코드 수행
  }
  ```
- **주의할 점**: `unordered_map`은 내부적으로 빠른 탐색을 위해 해시(hash)를 사용합니다. 따라서 키를 추가하거나 찾는 연산이 평균적으로 매우 빠릅니다.

### 디버그 로그와 작업 완료 확인
생성되는 분석기 템플릿은 `--debug` 옵션을 통해 디버그 로그를 켤 수 있습니다. 이를 활용하면 다음과 같은 정보를 쉽게 확인할 수 있습니다.

- 현재 몇 번째 이벤트를 처리 중인지
- 입력 파일이 몇 개인지, 각 파일의 처리가 끝났는지
- 최종적으로 생성된 출력 파일과 저장된 히스토그램 요약

디버그 모드가 아니더라도, 중요한 단계마다 "어떤 파일을 읽었는지", "전체 이벤트 처리가 끝났는지" 등을 알려주는 요약 로그가 출력되도록 설계되어 있습니다. 이렇게 하면 긴 배치 작업을 돌린 뒤에도 결과가 정상적으로 끝났는지 안심하고 확인할 수 있습니다.

### ROOT6/ROOT7 대비를 위한 현대적인 작성법
코드는 ROOT6 이상 환경에서 권장하는 `rootcling`을 사용해 dictionary를 생성합니다. 이는 클래스 정보를 ROOT에 알려 주어, 트리가 해당 클래스 객체를 저장하거나 읽을 수 있게 하기 위해 필요합니다. 분석에 맞춰 dictionary가 더 이상 필요하지 않다면 `Makefile`에서 해당 규칙을 제거해도 되지만, 사용자 정의 클래스(예: `eventBuffer`)를 ROOT I/O와 함께 쓰고 싶다면 유지하는 것이 안전합니다.

생성되는 C++ 템플릿은 다음과 같은 현대적인 C++ 문법을 사용합니다.

- `auto`와 범위 기반 for문(`for (auto &entry : container)`)으로 가독성을 높임
- `std::unique_ptr` 등 스마트 포인터 사용으로 메모리 누수를 예방함
- 명시적인 `#include`와 네임스페이스 사용으로 ROOT6/ROOT7 환경에서도 깨끗하게 빌드될 수 있도록 정리함

### 더 알아보기
- 분석기를 처음 작성한다면 `variables.txt`를 만들기 위해 `bin/mkvariables.py`를 먼저 실행해 보세요.
- 생성된 C++ 분석기는 `make`를 통해 컴파일하고, Python 분석기는 바로 실행할 수 있습니다.
- ROOT 환경 설정이 필요하므로, `setup.sh` 혹은 개인이 사용하는 ROOT 환경 설정 스크립트를 먼저 실행하는 것을 권장합니다.

이 가이드는 초보자도 treestream의 구조를 빠르게 이해하고, 현대적인 ROOT 분석 코드를 작성하는 데 도움을 주기 위한 것입니다.
