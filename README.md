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

### 핵심 파일별 역할
- **`bin/mkanalyzer.py`**: `variables.txt`를 읽어 C++/Python 분석기, `eventBuffer` 헤더, 링크 정의, Makefile, README까지 한 번에 생성합니다. 템플릿 문자열 안에 실제 코드 조각을 모아두고, 사용자가 지정한 출력 디렉터리에 필요한 파일을 모두 써 줍니다.
- **`bin/mklist.py`**: 입력 루트 파일에서 사용 가능한 트리와 브랜치를 훑어 보고, 어떤 변수를 `variables.txt`에 담을지 미리 확인할 수 있는 간단한 나열 도구입니다.
- **`bin/mkvariables.py`**: 지정한 ntuple을 열어 브랜치 정보를 추출한 뒤, `mkanalyzer.py`가 읽을 `variables.txt`를 자동으로 만들어 줍니다.
- **`include/pdg.h` · `src/pdg.cc`**: PDG ID를 사람이 읽을 이름으로 바꿔 주거나, Monte Carlo 입자 계보를 출력하는 유틸리티 함수 모음입니다. 분석기에서 입자 정보를 해석할 때 바로 쓸 수 있게 독립 모듈로 제공됩니다.
- **`include/treestream.h` · `src/treestream.cc`**: `itreestream`/`otreestream` 클래스를 정의하여 ROOT 트리를 C++ 객체처럼 순회하거나 새 트리를 저장할 수 있는 핵심 I/O 계층입니다. `eventBuffer`가 실제 데이터를 불러오거나 쓰는 기반이 됩니다.
- **`tnm/tnm.h` · `tnm/tnm.cc`**: 분석기 실행에 필요한 공용 유틸리티(명령줄 파싱, 출력 파일 관리, 자주 쓰는 수학 함수 등)를 모았습니다. `mkanalyzer.py`가 만들어 주는 분석기 본문에서 바로 include 하도록 설계되어 있습니다.

### `mkanalyzer.py`가 뼈대 코드를 엮는 방법
1. `variables.txt`를 파싱해 각 브랜치의 타입·길이·카운터 정보를 수집합니다.
2. 위 정보를 `eventBuffer` 템플릿(`TEMPLATE_H`, `TEMPLATE_CC`)에 채워 넣어, `treestream.h`의 스트림 클래스와 연동되는 버퍼 클래스를 생성합니다.
3. 생성되는 분석기 C++ 파일은 `tnm/tnm.h`를 include 하여 `commandLine`, `outputFile`, `fileNames` 등 공용 함수를 재사용하고, 앞서 생성된 `eventBuffer.h`를 통해 트리 데이터를 읽습니다.
4. Python 템플릿 역시 ROOT 파이썬 바인딩에서 `tnm` 함수와 `eventBuffer` 클래스를 불러와 동일한 흐름을 따르도록 구성되어 있습니다.
5. Makefile 템플릿은 `treestream.cc`, `pdg.cc` 등의 라이브러리를 링크하고, 필요하면 ROOT dictionary를 만들도록 규칙을 포함합니다. 설치 후에는 `include/`와 `src/`의 코드가 그대로 재사용됩니다.

### 기존 코드와 이전 제안의 차이
- 원본 `mkanalyzer.py`는 브랜치 선택 토글을 `std::map`으로 구현하고, 이벤트 루프는 최소한의 로그만 제공합니다.
- 이전 제안에서는 이 부분을 `std::unordered_map`과 추가 디버그 인자로 확장하려 했으나 구조 자체는 크게 바뀌지 않았습니다.
- 현재 저장소에는 원본 흐름을 복원하고, 향후에 직접 개선할 수 있도록 **한국어 주석**으로 개선 포인트(예: `std::unordered_map`으로 교체, 디버그 로그 추가, dictionary 규칙 끄기)를 표시해 두었습니다.

### PhysicsTools/TheNtupleMaker 의존성 정리
- `src/treestream.cc`, `src/pdg.cc`, `tnm/tnm.h` 등은 `#include "PhysicsTools/TheNtupleMaker/interface/..."`를 먼저 시도한 뒤, 같은 파일을 로컬 경로에서 다시 include 합니다. 이는 CMSSW 안에서 사용할 때 기존 모듈과 충돌하지 않도록 하기 위한 **호환성 장치**입니다.
- 이 저장소만으로도 빌드가 가능하도록 로컬 헤더가 바로 이어서 include 되므로, CMSSW 환경이 아니라면 추가 의존성은 없습니다.
- 만약 해당 경로를 완전히 없애고 싶다면, 위 파일들의 `#ifdef PROJECT_NAME` 블록을 정리하고, Makefile/컴파일 옵션에서 `-Iinclude` 만 남겨도 됩니다. README와 `bin/mkanalyzer.py`의 주석에 dictionary 규칙을 해제하는 방법도 함께 안내되어 있습니다.

### 향후 수정 포인트 안내
- `bin/mkanalyzer.py`의 템플릿 안에 `// NOTE(KR): ...` 또는 `# NOTE(KR): ...` 형태의 주석을 추가했습니다. `std::map` 대신 `std::unordered_map`으로 바꾸거나, 디버그 로그/진행률 출력, dictionary 생성 비활성화 등을 적용하고 싶을 때 해당 위치를 찾아 수정하면 됩니다.
- Python 템플릿에도 같은 위치에 주석이 있으니, C++과 Python 분석기를 동시에 유지하려면 두 곳을 함께 손보면 됩니다.
- 필요한 변경을 직접 시도하면서 README를 참고하면, NanoAOD뿐 아니라 miniAOD 등 다른 ntuple 구조에도 맞게 뼈대를 확장하기 수월해집니다.
