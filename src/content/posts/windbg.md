---
title: Kernel Debugging with WinDbg
published: 2024-06-19
description: 'Let us capture bugs in kernel with Windbg'
image: ''
tags: [ WinDbg ]
category: 'Reversing'
draft: false 
---

## WinDbg?

리버싱을 하면서 다들 [IDA](https://hex-rays.com/), [Ghidra](https://www.ghidra-sre.org/)를 사용해본 경험이 있을 것이다. 이때 리버싱에는 디버깅이 필수적으로 따라붙게 되는데, 이때 많은 사람들이 사용하는 디버거는 [gdb](https://www.sourceware.org/gdb/)이다. 지금 소개할 것은 이와 비슷한 [WinDbg](https://learn.microsoft.com/ko-kr/windows-hardware/drivers/debugger/)이다.

WinDbg는 마이크로소프트에서 만든 'Windows Debugger'로, 윈도우를 설치할 때 몇 가지 툴과 함께 딸려오는 디버거이다.

gdb가 Linux, Windows, 및 여러 프로그램에 대한 디버깅 도구를 제공한다면 WinDbg같은 경우 Windows에 특화된 디버거라고 할 수 있다.

유저 모드 프로그램 (.exe 파일)의 경우 gdb와 WinDbg의 성능 차이가 별로 안 나 보일 수 있겠다. 그도 그럴 게 WinDbg는 사용법이 좀 까다롭고, 그 점이 내가 이 글을 쓰는 이유이다. 하지만 실제로도 PE 파일에 대한 리버싱은 WinDbg가 이점을 가지지 못한다. 그러면 왜 WinDbg를 쓰냐? 주로 윈도우 커널 디버깅을 수행할 때 자주 쓰이는 것이 WinDbg이다.

## How?

커널 디버깅을 수행하기 위한 두 가지 방법이 존재한다.

### Local Kernel Debugging

우선 첫 번째 방법으로는 **Local Kernel Debugging**이 있다. 말 그대로 자기 컴퓨터를 갖다가 커널로 디버깅 하는 것이다. 하지만 이는 심각한 제한 (예를 들어 bp를 설치를 못한다.)이 동반된다. 하지만 하는 방법은 쉽다. 그냥 커널을 들여다 보고 싶을 때 쓰는 방법이다.

:::note[Local Kernel Debugging]

**Start Debugging -> Attach to Kernel -> Local**

:::

<img src="/windbg/windbg.png">

### Remote Kernel Debugging

두 번째 방법으로 추천하는 방식인 **Remote Kernel Debugging**이 있다. 이 방식의 경우 다른 컴퓨터에 접속하여 그 커널을 디버깅하는 방식으로, 보통 VM을 돌린다. (호환성이 좋은 VMware을 사용한다.)

RKD를 하려면 다음 단계를 수행한다.

:::note[Remote Kernel Debugging]

**Debuggee OS -> `kdnet`** 으로 어떤 네트워크를 통해 디버깅 소켓을 열 수 있는지 확인한다. 

:::

`kdnet` 은 보통 `C:/Program Files (x86)/Windows Kits/10/Debuggers/x64/` 에 존재한다.

<img src="/windbg/kdnet.png">

:::note[Remote Kernel Debugging]

**Debuggee OS -> `kdnet [Debugger OS' IP]`** 로 서버를 연다. 확인한 네트워크의 **호스트 주소**를 바탕으로 열어야 한다.

:::

<img src="/windbg/kdnet_2.png">

:::note[Remote Kernel Debugging]

**Debugger OS -> WinDbg -> Attach to Kernel -> Net** 에서 서버로 접속한다. 2단계에서 출력된 명령어를 실행할 수 있지만 그러면 구 버전 디버거가 실행되어 디버깅 퍼포먼스가 약간 떨어진다.

:::

<img src="/windbg/kdnet_3.png">

:::note[Remote Kernel Debugging]

**Debuggee OS -> Reboot** 으로 컴퓨터를 재부팅한다. 재부팅하면 부팅 단계에서 WinDbg가 Attach된다.

:::

## Features

1. `CTRL+C` 나 상단의 Break 버튼을 눌러서 그 즉시 Debuggee에 BP를 걸 수 있다.

2. 여러 명령어를 통해 아주 다양한 행위를 수행할 수 있다.

3. DSE가 종료된다. 그 시점부터 서명되지 않은 드라이버도 로드될 수 있다.

4. KPP (PatchGuard)의 추가적인 플래그가 설정된다. 이는 gdb로 디버깅했을 때는 설정되지 않는 플래그이다.

명령어를 알아보자. 많이 쓰이는 순서대로 작성하겠다.

| 명령어         | 예제                                                      | 설명                                                              | 유사 명령어                                                                                            |
| ----------- | ------------------------------------------------------- | --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| `g`         |                                                         | `Go`. 함수의 실행을 계속한다. gdb의 `c` 와 같다.                              |                                                                                                   |
| `t`         |                                                         | Step Into. `F11` 로도 가능하다.                                       |                                                                                                   |
| `p`         |                                                         | Step Over. `F10` 로도 가능하다.                                       |                                                                                                   |
| `gu`        |                                                         | Step Out. `Shift+F11` 로도 가능하다.                                  |                                                                                                   |
| `dt`        | `dt _EPROCESS 0xffffdeadbeef`                           | 주소를 특정 struct로 변환한다. **정말 유용하다.**                               | `du` (Unicode String), `da` (ANSI String)                                                         |
| `bp`        | `bp 0xffffcafebabe`                                     | 브레이크 포인트를 삽입한다.                                                 | `bl` (bp list), `bc` (bp remove), `bd`/`be` (bp dis/enable), `bm` (bp @symbol), `ba` (Watchpoint) |
| `k`         |                                                         | Call Stack을 표시한다.                                               | `k [n]`으로 최대 n개의 함수를 출력할 수 있다.                                                                    |
| `lm`        | `lm m nt*`                                              | 로드된 모듈을 표시한다.                                                   | `lm v`로 모듈의 상세 정보를 출력할 수 있다.                                                                      |
| `eb`        | `eb .-4 57 61 6E 65`                                    | `eb .-[n] {list}` 로 앞 n개의 바이트를 수정할 수 있다.                        | `a`                                                                                               |
| `x`         | `x nt!*`                                                | `eXamine`. 무언가를 찾는다.                                            |                                                                                                   |
| `.symfix`   |                                                         | 마이크로소프트 심볼 정보를 로드한다.                                            | `.reload` 로 리로드 하자.                                                                               |
| `.writemem` | `.writemem C:/Users/wane/dump.bin 0xffff12345678 L6974` | `.writemem [File] [Address] L[Size]`. Dump와 같은 기능을 한다.          | `.dump`                                                                                           |
| `dds`       | `dds esp esp+100`                                       | `Display Words and Symbols`. 보통 콜 스택이 깨져서 `x` 명령을 쓰지 못할 때 사용한다. | `dds [Options] [Start] [End]`                                                                     |
| `q`         |                                                         | 디버깅 종료. 근데 이거 쓸 때는 항상 빡종할 때이다.                                  | `qd` (연결 해제)                                                                                      |



이 외에는 WinDbg의 GUI를 사용하면서 익힐 수 있다. GUI에서 Memory View, Register View/Edit, Memory Patch 등을 아주 쉽고 시각적으로 할 수 있기 때문에 따로 명령어를 익힐 필요가 없다.

하지만 `lm`, `.writemem` 과 같은 명령어는 CLI 환경에 특화된 명령어이기 때문에 이런 것을 외워둬야 한다.

## Conclusion

WinDbg는 윈도우 커널을 조사할 때 사용되는 아주 강력한 도구이다. 이 도구를 사용하면 Windows의 Undocumented 함수들, 수상한 드라이버, 안티치트 리버싱 등에서 강력한 이점을 가질 수 있다.

하지만 Scylla같은 제3자 모듈을 로딩할 수 없고 PatchGuard를 분석하기에 별로 좋지 않은 툴인 점을 보아 리버서들이 쓰기보다는 시스템 개발자들이 쓰는 경우가 많다. (보통 고수 리버서들은 안 걸리는 [HyperDbg](https://hyperdbg.org/)를 쓴다.)

그럼에도 WinDbg는 아주 대중적이고 강력한 툴이며 배울 가치가 있어서 소개하게 되었다.
