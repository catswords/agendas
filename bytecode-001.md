# bytecode-001

어셈넥스트는 산업계의 요구사항에 맞추기 위해 웹어셈블리(WASM)에 국한하지 않고 바이트코드 범위에 속하는 아래의 기술들을 논의할 수 있도록 운영하고 있습니다.

## 바이트코드 종류
  * WebAssembly (WASM)
  * Java bytecode (자바의 경우 JNI까지 포함)
  * MSIL (C# 등 닷넷 환경에서 사용)
  * P-Code(VBA)
  * Dalvik bytecode(Android)
  * Industrial IL (IEC 61131-3, 주로 PLC에서 사용)
  * LLVM
  * The custom (virtualized) machine code
  * x86-64/ARM/MIPS/RISC-V assembly (네이티브)

## 다양한 언어를 WebAssembly에

### 개요

우리는 반세기 전 사용되던 컴퓨터 프로그래밍 언어(코볼, 포트란, 파스칼, 에이다, 스몰토크 등)의 존재를 학부 과정에서 배웁니다.

이 중 일부 언어는 신생 언어 뒤에 숨어 현재까지도 널리 사용하고 있습니다. 일부 파이선(Python)의 연산 라이브러리에는 포트란(Portran)이 사용됩니다.

60년대 프로그래밍을 대표하는 코볼(Cobol)은 2019년 COVID-19 상황으로 인해 증가한 미국의 국가 행정 수요를 감당하지 못해, 2020년 IBM은 대학가를 중심으로 코볼 교육을 다시 시작했습니다.

하지만 과거의 언어들은 최근의 언어가 제공하는 편의성과 시스템 호환성 모두에서 밀려나 있습니다. 그럼에도 최근의 언어와 그 생태계가 만족시키지 못한 많은 유형의 요구를 감당하고 있습니다.

여기서는 WebAssembly (일명 WASM)으로 최신의 시스템에서 과거의 언어를 사용하기 위한 자료를 정리해두었습니다. 

### COBOL

#### cloudflare/cobweb

이 프로젝트를 사용하면 COBOL 프로그램을 웹어셈블리로 컴파일할 수 있습니다.

  * https://github.com/cloudflare/cobweb

### FORTRAN

#### FORTRAN In The Browser

이 기사는 Fortran(F90)을 웹어셈블리로 컴파일 하기 위한 툴체인(LFortran, Flang-7, F18, Dragonegg, AOCC 등)을 다룹니다.

  * https://chrz.de/2020/04/21/fortran-in-the-browser/

그 외 자료

  * https://github.com/DirkWillem/WebAssembly-Fortran-Demo
  * https://github.com/flang-compiler/flang

### Pascal

#### FreePascal

아래 글들은 Pascal을 웹어셈블리로 컴파일 하기 위한 방법을 안내합니다.

  * https://wiki.freepascal.org/WebAssembly/Compiler

#### Oz.Wasm

이와 반대로 Pascal로 짜여진 WASM 인터프리터가 존재합니다. 이것은 기존 Pascal 언어 사용자가 WASM의 특성을 이해하는데 도움이 될 수 있습니다.

  * https://github.com/marat1961/wasm

### ADA

#### godunko/adawebpack

이 프로젝트를 사용하면 ADA 언어로 작성된 프로그램을 웹어셈블리로 컴파일 할 수 있습니다.

  * https://github.com/godunko/adawebpack
  * https://blog.adacore.com/use-of-gnat-llvm-to-translate-ada-applications-to-webassembly

### SmallTalk

#### Squeak

스몰토크는 자바(Java)와 유사하게 가상머신(VM) 방식으로 작동하는 객체지향 프로그래밍 언어입니다. 해당 VM인 Squeak의 자바스크립트 버전인 SqueakJS가 존재하지만 이것을 WASM으로 포팅한 저명한 실제 사례는 아직 없습니다.

[관련 이슈](https://github.com/codefrau/SqueakJS/issues/61)가 한차례 제기된 적이 있습니다. 이론적으로는 툴체인 구성이 가능한 기술 스펙으로 이루어져 있습니다.

  * https://squeak.js.org/
  * https://squeak.org/

### 그 외: Erlang/Elixir

#### Lumen

Erlang 및 Elixir는 최근에도 활발한 유지보수가 되고 있는 언어이지만 주요 사용자 분포는 위에 언급한 언어와 크게 다르지 않습니다. Lumen은 BEAM 가상머신에서 돌아가는 언어 계열(Erlang 등)을 웹어셈블리로 컴파일 하기위한 프로젝트입니다.

  * https://getlumen.org/
  * https://github.com/lumen/lumen

### Java

#### JVM(또는 Java Bytecode)을 이용하는 WASM 지원
* https://github.com/cretz/asmble
* https://teavm.org/

#### Java 바이트코드를 WASM으로 지원
* https://github.com/i-net-software/JWebAssembly
* https://github.com/mirkosertic/Bytecoder

#### WASM 기반 가상머신 방식
* https://medium.com/leaningtech/cheerpj-2-1-released-java-bytecode-to-webassembly-and-javascript-303fb8dd5d98

#### wasmer를 이용한 방법 (추천)
* https://github.com/wasmerio/wasmer-java

#### 한국 환경에 맞춘 좋은 접근법?
* 인기 프레임워크(Spring/eGovFrame 등)와 결합하여 시너지를 낼 수 있는 방법?
* 그 외 생각나는데로 추가
