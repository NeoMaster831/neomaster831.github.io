---
title: Windows 11 24H2 Kernel Patch Protection Analysis
published: 2025-09-09
description: "Let's bypass PatchGuard in Windows 11 24H2, on-the-fly."
image: ''
tags: [ 'Windows', '']
category: 'Windows'
draft: false 
---

# Windows 11 24H2 Kernel Patch Protection

[GitHub](https://github.com/NeoMaster831/kurasagi)에 개인 프로젝트를 진행하면서 같이 [pdf](https://github.com/NeoMaster831/kurasagi/tree/product) 또한 올렸는데, 생각해보니 이를 정리한 글이 하나 쯤은 있어야 한다고 생각하였습니다. 같은 내용을 GitHub에서 확인하실 수 있습니다.

정말 긴 여정이 될 것입니다. 준비하시고 읽는 것을 추천드립니다.

### Abstract

본 연구는 Windows Kernel Patch Protection(PatchGuard, PG)에 대한 심층 분석을 수행하고, 나아가 최초로 최신 Windows 11 24H2 버전에 구현된 PG의 내부 구조와 동작 방식을 완전 분석 및 우회한 사례를 제시한다. 본 연구는 최신 빌드 환경에서의 PG 초기화 절차, 암호화·난독화 기법, 감지·우회 로직까지 전 과정을 체계적으로 규명하였다. 이를 통해 PG의 핵심 구조를 실시간으로 추적·해체하는 기술을 확립하고, 베어메탈 환경에서 동작하는 완전한 우회 기법을 구현함으로써 보안 연구와 방어 전략 수립에 결정적인 통찰(Mitigation Insight)을 제공한다.

# Disclaimer

**이 연구는 오직 교육적 목적으로 이루어졌습니다.**

## Environment

사용된 소프트웨어 및 하드웨어는 다음과 같습니다.
+ Windows 11 Home 24H2, Build 26100.4351 (Bare-Metal)
+ Windows 11 Pro 23H2 (VM)
+ 12th Gen Intel(R) Core(TM) i7-12700H
+ VMware Workstation Player 17

# Introduction

KPP(Kernel Patch Protection)는 Windows XP x64 (2005) 때부터 추가된 커널 패치 보호 매커니즘으로, PG(PatchGuard)라고도 알려져 있습니다. 이 문서에서는 이후 PG라는 용어를 사용합니다.

PG는 이름에서도 알 수 있듯이 기본적으로 Windows NT 커널에 행해지는 모든 불법적인 패치나 변조를 보호합니다. 이는 단순히 코드 패치 뿐만 아니라 DKOM과 같은 커널 내부 구조체를 직접적으로 수정하는 것을 금합니다.

PG는 구체적으로 다음과 같은 시스템 요소를 보호하며, 더 자세한 사항은 [Microsoft의 문서](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/bug-check-0x109---critical-structure-corruption)에 상세히 기술되어 있습니다.
+ NT 커널의 모든 함수
+ 프로세서의 GDT, IDT, DR7 등 민감한 레지스터
+ NT 커널의 오브젝트 및 구조체 (`EPROCESS`, SSDT, 여러 전역 변수)
PG는 시스템이 무결성 검사를 통과하지 못한다고 판단될 즉시 0x109번 BSOD (`CRITICAL_STRUCTURE_CORRUPTION`) 를 유발하여 악성 코드가 메모리에 상주하지 못하게 막는 것이 기본적 절차입니다.

PG는 공격자에게 지속적인 제약과 우회 시도 시 탐지 리스크를 남겨 루트킷의 활개를 효과적으로 저지하였습니다. 이러한 이유로 대부분의 악성 코드 및 게임 치트 프로그램은 커널 오브젝트에 직접적으로 데이터를 기록하지 않거나, NT 커널의 핵심 함수에 대한 후킹을 회피하는 등 PG를 철저히 고려하여 설계됩니다.

## Things can be caused if PG is disabled

PG가 무력화될 경우 커널 루트킷이 상주할 수 있게 됩니다. 이후 Direct Kernel Object Manipulation을 통한 커널 오브젝트 변조로 보안 제품과 시스템 정책을 마음대로 수정하고 변질시킬 수 있습니다. 이 중 가장 위험한 것은 Windows ETW의 완전한 장악입니다. PG로 인해 보호받던 ETW는 이제 안전하지 않으며, 변조, 삭제에 취약해지게 됩니다.

예를 들어, ETW 관련 커널 함수를 후킹해 ETW 이벤트를 필터링하거나 드롭하여 SIEM에 도달하는 로그를 선택적으로 검열할 수 있습니다. 분석자는 ETW가 비활성화된 이후 어떤 식으로 공격을 지속적으로 감행했는지 알 수 없습니다.

ETW 로그 공백 때문에 “어디까지 번졌는가?”를 정량화할 수 없습니다. 스코핑이 축소되어 잔존 비컨 및 백도어가 남고, 이후 재감염이 *새로운 사고처럼* 관측됩니다. 이로 인해 결과적으로 침해 사고에 대한 디지털 포렌식이 힘들어집니다.

이 외에도 커널 취약점의 실현이 더 쉬워지고(DKOM의 지속 가능성 확보), 모든 커널 모듈의 신뢰도가 급격하게 떨어지게 됩니다. 각종 보안 솔루션은 윈도우 커널 모듈의 신뢰도를 바탕으로 동작하는 경우가 많은데, 이렇게 PG 우회로 인해 신뢰도가 감소하면 그 위에서 동작하는 보안 솔루션 전반에 치명적인 영향을 줍니다.

## Motivation

공격자들은 대부분 PG를 우회할 수 없다고 생각하기 때문에, PG와의 정면 충돌을 회피합니다. 하지만 일부 고수준의 해커는 PG를 우회하고 심각한 위협을 일으키는 APT 기법을 개발합니다.

본 연구의 목적은 단순한 PG 분석이나 우회 기술의 구현에 있지 않고, 고수준의 공격자들이 활용하는 내부 매커니즘을 정밀하게 이해한 뒤 **방어적 통찰**을 제공하는 데 핵심 의도가 있습니다.

분석을 진행한 저 또한 PG 우회자들과 동등한 기술력을 지니기 위해 프로젝트를 진행하게 되었습니다.

## Evolution of PG

PG는 지금까지 끊임없이 진화해왔습니다. 초기 Windows XP 및 Vista의 PG는 BSOD를 유발하는 함수인 \texttt{KeBugCheckEx}를 직접 후킹하여 우회할 수 있는 단순한 수준이었으나, Windows 7부터는 함수 진입 전에 미리 체크섬을 검증하는 방식으로 이러한 우회 기법을 막았습니다. 이후 Windows 8 이상에서는 PG 자체의 체크섬을 지속적으로 검증하는 등 더욱 정교해지면서 우회가 어려워졌으며, Windows 11에서는 가상화 기술의 본격적인 도입과 함께 HyperGuard(HG)가 등장하여 한층 더 견고한 보호 기법이 구현되었습니다.

지금까지 다양한 연구자들이 PG 대한 분석 자료를 공개해왔지만, 대부분 수 년 전 작성된 것으로 최신 Windows 11 버전에 대한 정보는 거의 존재하지 않습니다. 이에 따라, 저는 작성일 기준으로 가장 최신 빌드인 Windows 11 24H2를 대상으로 PG 분석을 직접 진행하였습니다.

# Analysis

과거부터 PG 분석이 활발히 이루어져 왔습니다. 저는 그런 과거 연구 자료를 참고해 가며 기본적인 정보를 숙지한 뒤 분석을 시작하였습니다.

PG는 분석가들에게 혼란을 주기 위해 아예 다른 이름으로 심볼화되어 있거나, (e.g. `KiFilterFiberContext`) 심볼이 없습니다. 교묘하게 정상적인 루틴 내에 숨겨진 PG 루틴이 있는 경우 또한 있습니다.

## Init Trigger, KiFilterFiberContext

기본적으로 PG는 `ntoskrnl.exe`가 초기화될 때 같이 초기화가 이루어집니다. 여러 NT 모듈이 비밀스럽게 PG Init을 호출합니다.

<img src="/pg_bypass/PGInit.png">

PG를 초기화하는 함수는 `KiInitializePatchGuard`로 알려져 있는데, 해당 함수를 호출하는 다른 함수 중 가장 중심적인 함수는 `KiFilterFiberContext`입니다. 즉, 직접적인 초기화는 `KiInitializePatchGuard`에서 수행하지만, 이를 준비하고 트리거하는 'Stub' 함수 중 가장 중요한 함수가 `KiFilterFiberContext`입니다. 따라서 이 함수는 PG 분석에 있어 매우 중요한 함수 중 하나입니다. 이 외에도 여러 다른 함수가 있으나, 가장 자주 호출되고 눈여겨볼만한 함수는 `KiFilterFiberContext`이기 때문에, 먼저 `KiFilterFiberContext` 함수를 조사해보겠습니다.

지금부터 제시하는 코드들은 모두 컨텍스트가 매우 크기 때문에, 의사 코드로 변형하여 제공하겠습니다. 매우 큰 컨텍스트를 옮기는 과정에서 의도하지 않은 손실이나 비약이 있을 수 있습니다. 제가 작성한 의사 코드는 언어적으로 이해하여 전체적인 흐름을 파악하는 용도로 사용해 주시기 바랍니다.

```pseudo
Algorithm: KiFilterFiberContext

Input:  FIBER_CONTEXT_PARAM fiberParam
Output: BOOLEAN result

1. AntiDebug
2. If MaxDataSize not initialized:
     - If PsIntegrityCheck:
         - create callback "542875F90F9B47F497B64BA219CACF69"
         - notify callback
3. r1 ← rand(0, 12)
4. r2 ← rand(0, 5)
5. r3 ← rand(0, 9)
6. result ← KiInitializePatchGuard(r1, r2, r3 ≤ 5, fiberParam, 1)
7. If result:
     - If r1 ≤ 5:
         - r4 ← rand(0, 12)
         - r5 ← rand(0, 5) (≠ r2)
         - result ← KiInitializePatchGuard(r4, r5, r3 ≤ 5, fiberParam, 0)
     - If result:
         - result ← KiInitializePatchGuard(0, 7, 1, 0, Unknown)
8. AntiDebug
9. ClearTails
10. Return result
```

알고리즘을 자세히 보면, `KiInitializePatchGuard` 함수를 랜덤한 파라미터로 호출합니다. 즉, 해당 `KiFilterFiberContext` 함수는 `KiInitializePatchGuard`를 호출하는 발사대 역할을 하는 함수임을 알 수 있고, `KiInitializePatchGuard` 함수는 여러 번 호출될 수 있음을 시사합니다.

ClearTails의 경우, 초기화가 끝나고 런타임에 PG 관련된 컨텍스트를 메모리에서 찾을 수 없게 만드는 코드입니다. 분석에 있어 중요한 부분이 아니기 때문에 간단하게 한 줄로 나타내었습니다. fiberParam은 나중에 Method 3 패치 가드 초기화(섹션 3.2.10.4)에서 쓰입니다. 해당 파라미터는 `NULL`일 수 있으며, `NULL`인 경우가 대부분이지만 아닌 경우 또한 있습니다. (섹션 3.1.2.2)

`MaxDataSize`의 경우 앞으로 설명할 PG 글로벌 포인터입니다. 이 변수는 나중에 전역 PatchGuard 컨텍스트(섹션 3.2.8)에 사용되는 전역 변수입니다. `rand` 함수는 $[a, b]$ 구간에 대해 랜덤을 생성하는 함수입니다.

### Where is Callback!?

알고리즘을 자세히 보면 초반에 콜백과 관련된 부분이 있는데, 실제로 해당 부분의 디컴파일된 C 의사코드를 보면 다음과 같이 되어 있습니다.

```c++
ObjectAttributes.Length = 48;
ObjectAttributes.ObjectName = (PUNICODE_STRING)L"TV";
ObjectAttributes.RootDirectory = 0LL;
ObjectAttributes.Attributes = 64;
*(_OWORD *)&ObjectAttributes.SecurityDescriptor = 0LL;
if ( ExCreateCallback(&CallbackObject, &ObjectAttributes, 0, 0) >= 0 )
{
    ExNotifyCallback(CallbackObject, PgTVCallbackDeferredRoutine, &__24);
    ObfDereferenceObject(CallbackObject);
    if ( __24 )
      __2c = 1;
    ExInitializeNPagedLookasideList(&stru_140E0EF80, 0LL, 0LL, 0x200u, 0xB38uLL, 0x746E494Bu, 0);
}
```

많은 연구자들이 `ObjectName`이 `"TV"`로 되어 있어 콜백 이름을 'TV 콜백'이라 칭하지만, 이것은 디컴파일러의 오류로 인한 오해입니다. 자료형이 `UNICODE_STRING`이기 때문에 실제로 콜백 이름은 다릅니다.

실제 이름은 `"\Callback\542875F90F9B47F497B64BA219CACF69"` 입니다. 별개로 'TV 콜백'이라는 단어가 더 보편적이기 때문에, 본 문서에서는 이후로 'TV 콜백'이라는 단어를 사용합니다.

커널 콜백은 대부분 `ExCreateCallback`, `ExRegisterCallback`, `ExNotifyCallback` 순서의 시나리오를 따릅니다. `ExCreateCallback`에서 생성된 콜백을 `ObjectName`을 통해 `ExRegisterCallback`에서 등록한 뒤, `ExNotifyCallback`을 통해 등록된 모든 콜백에 대해 함수를 호출합니다. `ExCreateCallback`과 `ExRegisterCallback`의 순서는 큰 영향을 끼치지 않습니다.

하지만 코드를 보면 `ExRegisterCallback`은 어디에도 보이지 않습니다. 즉, 어디선가 이미 `ExRegisterCallback`을 사용한 것으로 보입니다. 사실 어디서 호출하였는지는 중요하지 않습니다. `ExNotifyCallback`을 통해 호출되는 루틴, 즉 `PgTVCallbackDeferredRoutine`이 중요합니다. (이것이 실제 검사에 사용되는 루틴이기 때문입니다.) 이 함수는 섹션 3.3.3 에서 자세히 확인할 수 있습니다.

### References

`KiFilterFiberContext` 함수를 호출하는 여러 모듈들에 대해 알아보겠습니다. 전에도 언급했듯, 여러 모듈에서 다양하고 교묘한 방법을 통해 호출합니다.

#### KeInitAmd64SpecificState

이 함수로 인해 항상 확정적으로 `KiFilterFiberContext`가 호출됩니다. 이름만 보면 언뜻 AMD64 CPU에 대해 특정적인 작업을 수행하는 것 같습니다. 실제로 이 함수는 `PipInitializeCoreDriversAndElam`라는 정상적인 함수에 자연스레 끼어 있습니다.

```
Algorithm: KeInitAmd64SpecificState

Input:  None  
Output: None  

1. If (KdDebuggerNotPresent and KdPitchDebugger):
     - KiFilterFiberContext(0)
```

하지만 실제로는 `KiFilterFiberContext`만을 호출하는 함수입니다. 이 함수는 디버깅 환경이 아닌 경우에 대해 Exception Handler를 통해 ZeroDivisionError를 일부러 발생시킨 뒤 핸들링하는 것으로, 언뜻 보면 아무것도 하지 않는 루틴으로 보여 분석가들을 혼란에 빠트립니다.

<img src="/pg_bypass/KeInitAmd64.png">

여기서 언뜻 추론할 수 있는 사실은, PG는 디버그 환경일 경우 초기화를 하지 않는다는 것입니다. 이는 분석가들을 방해하려는 의도가 있고, 또한 커널 모듈 개발자들이 드라이버를 개발할 때 PG 모듈과 충돌을 일으키지 않게 하려는 의도가 있습니다.

#### ExpLicenseWatchInitWorker

이 함수는 `ExpWatchProductTypeInitialization` 함수에서 호출됩니다.
`ExpWatchProductTypeInitialization` 함수는 Microsoft 라이선스를 검증하는 함수들이 많은 곳으로, 이 함수 역시 같은 부류인 함수인 척 하며 자연스럽게 PG를 초기화합니다.

```
Algorithm: ExpLicenseWatchInitWorker

Input:  None  
Output: None  

1. fiberParam ← KPRCB.HalReserved[6]  
2. If (rand(0, 99) < 4 and KdDebuggerNotPresent and KdPitchDebugger):  
     - KiFilterFiberContext(fiberParam)
```

이 함수의 경우 4%의 확률로 fiberParam을 동반하여 `KiFilterFiberContext`를 호출합니다. fiberParam의 경우 미리 초기화된 CPU의 `PRCB.HalReserved[6]` 멤버를 이용합니다. 물론 초기화가 끝나면 해당 멤버의 흔적은 사라져 전달된 fiberParam을 확인할 수 없습니다.

fiberParam의 자료형인 `FIBER_CONTEXT_PARAM`은 다음과 같습니다.

```c++
struct FIBER_CONTEXT_PARAM {
    UCHAR Code[4]; // prefetchw byte ptr [rcx]; ret
    DWORD Unk0;
    PVOID PsCreateSystemThreadPtr; // Used in Method 3
    PVOID PsCreateSystemThreadDeferredStub; // Used in Method 3
    PVOID KiBalanceSetManagerPeriodicDpcPtr; // Used in Method 5
}
```

이 구조체는 나중에 `KiInitializePatchGuard` 함수에서 중요한 구조체로 자리합니다.

#### Extra: KiVerifyXcpt15

이 함수는 `KiFilterFiberContext`를 거치지 않고 `KiInitializePatchGuard` 루틴을 호출합니다. 이 함수는 본래 정상적인 역할을 수행하지만, 디버깅 상태가 아닐 경우 `KeInitAmd64SpecificState` 함수처럼 Exception Handler를 통해 PG 초기화 루틴을 호출합니다.

해당 함수는 `KiVerifyXcptRoutines` 함수 배열 안의 멤버이며 `KiVerifyScopesExecute` 함수가 해당 배열 안의 함수를 모두 호출하는 것으로 트리거됩니다.

```
Algorithm: KiVerifyXcpt15

Input:  Unknown  
Output: Unknown  

...  
1. If (KdDebuggerNotPresent and KdPitchDebugger):  
     - KiInitializePatchGuard(rand(0, 12), 1, 2, 0, 0)  
...  
2. Return Unknown
```

## Initialization: KiInitializePatchGuard

`KiInitializePatchGuard` 함수는 본래 심볼이 없는 함수로, `KiFilterFiberContext` 및 `KiVerifyXcpt15` 함수에서 `KeExpandKernelStackAndCallout` 함수를 동반하여 호출되는 함수입니다.

해당 함수는 `ntoskrnl.exe`의 함수 중 가장 길이가 긴 함수입니다. 실제로 디컴파일을 시도하면 약 32000줄의 코드가 등장합니다. 물론 이것은 난독화, 인라인 블록으로 인한 영향이 포함되어 있지만, 기본적으로 수행하는 일이 많은 함수입니다. 실제로 이 함수에서 PG 모듈에 대한 거의 모든 힌트를 획득할 수 있습니다. 따라서 이 함수를 잘 분석할수록 PG에 대한 이해도가 높아지고, 결과적으로 성공적인 우회를 이끌어낼 수 있습니다. 저 또한 이 함수에 큰 비중을 실어 분석하였습니다.

하지만 개발자들도 이 사실을 인지하고 이 초기화 함수에 각종 난독화 및 인라인화를 적용해 두었습니다. 이 때문에 해당 함수의 크기가 자그마치 C 의사코드 32000줄에 육박하게 되었고, 결과적으로 해당 함수를 분석하는 것이 매우 어려워졌습니다.

하지만 난독화를 한 코드를 보면 가상화가 적용되어 있지 않다는 사실을 알 수 있습니다. 가상화가 적용이 안 되어 있는 코드는 분석하기 상대적으로 쉽기 때문에, 끈기있게 분석을 하게 된다면 초기화 루틴을 대강 파악할 수 있습니다.

```
Algorithm: KiInitializePatchGuard

Input:
  DWORD dpcIndex, DWORD method, DWORD vUnk0,
  FIBER_CONTEXT_PARAM fiberParam, BOOLEAN largeCheck
Output: BOOLEAN result

--- Initialization Phase ---

1. If ("TESTSIGNING" and "DISABLE_INTEGRITY_CHECK" are in Boot Options and code patches are allowed):
     - return False

2. If (method == 5 and fiberParam == NULL):
     - method ← 0

3. crArr ← CriticalRoutines

4. PG_Context ← MemoryAllocate(size = 0x100000 + 0xAE8 + α)
5. PG_Context.AddRandomPadding()
6. PG_Context.Fill(CmpAppendDllSection's bytecode)

7. For f in NT Functions (used in this routine):
     - PG_Context.Fill(f)
     - // Disturb analyzers; behave like a tiny VM
     - // e.g., store ExAllocatePool2 at (PG_Context + 32) and call via (PG_Context + 32)(...)

8. PG_Context.Fill(PTEs of crArr)
   // Mitigation for "hooking KeBugCheckEx" — used to recover on BSOD trigger

9. For f in NT Functions:
     - PG_Context.AddDetection(f)

10. If (largeCheck):
      - For f in NT Functions (More):
           - PG_Context.AddDetection(f)

11. PG_Context.AddDetection(IDT)
12. PG_Context.AddDetection(GDT)

13. PG_Context.AddStructureDetections()

14. PG_Context.SetPerProcessorState()
15. PG_Context.SetCodeSectionInContext()
16. PG_Context.SetEntrypointProperly()
    // Set code section inside PG context itself
17. PG_Context.FinalizeCodeStuff()

--- Apply Phase ---

18. If (method == 3):
      - pgThread ← PG_Context.InitializePgThread()
      - PsCreateSystemThread(pgThread)

19. If (method == 4):
      - pgApc ← PG_Context.InitializePgApc()

20. If (method == 5):
      - PG_Context.InitializeHook(fiberParam)

21. If (method == 7):
      - InitializeGlobalContext(MaxDataSize, PG_Context)

22. PG_Context.finalize()
    // Final routine verifies itself and stored routines

23. PG_Context.EncryptSelf()
    // Encrypt self — ready to launch

24. pgDpc ← PgDpcRoutines[dpcIndex]

25. // Launch PatchGuard contexts; now ready to detect malicious routines
26. switch (method):
      case 0:
        KeSetCoalescableTimer(pgDpc)
      case 1:
        KPRCB.AcpiReserved ← pgDpc
      case 2:
        KPRCB.HalReserved[7] ← pgDpc
      case 4:
        KeInsertQueueApc(pgApc)
      case 3, 5, 7:
        Unknown

27. return True
```

저는 모든 루틴을 하나하나 정독하면서 분석하지 않고, 다양한 기법을 사용하여 필요한 부분만 발췌하여 분석하였습니다. 즉, 제가 조사했을 때 발견한 실제 검사에 영향이 미치는 것만을 의사 코드에 포함하였습니다. 이 방대한 루틴을 이와 같이 매우 짧은 알고리즘으로 이해하기에는 무리가 있으니, 서브섹션들에서 자세히 설명하겠습니다.

`KiInitializePatchGuard` 루틴을 크게 두 부분으로 나누게 되면, '초기화 부분'과 '적용 부분'으로 나눌 수 있습니다. 현재 환경 기준으로 C 의사코드 약 26000줄까지가 초기화 부분, 다음부터는 적용 부분입니다.

당연히 중요한 부분은 적용 부분입니다. 적용 부분을 자세히 파악할 수만 있다면, 아주 쉽게 우회할 수 있는 경우가 대부분입니다. 반대로 초기화 부분은 이해하면 도움이 되지만, 목표하는 '런타임 우회'의 관점에서 실행 흐름을 수정할 수 없기 때문에 우회하기 어렵습니다. 따라서 적용 부분에 힘을 실어 설명하겠습니다.

<img src="/pg_bypass/solve.png">

### PG Context

PG Context는 PG와 관련된 모든 변수, 코드, 검사 루틴 목록 등 다양한 컨텍스트를 저장하는 거대한 구조체입니다. 현재 빌드 24H2에서 측정된 크기는 자그마치 2792 (`0xAE8`) 바이트입니다.

중요한 멤버를 몇 가지 알아보겠습니다.

```c++
struct PG_Context {
    UCHAR CmpAppendDllSectionCode[sizeof(CmpAppendDllSection)];
    UnkSz0 vUnk0;
    struct _FnVm {
        PVOID strnicmpPtr;
        PVOID KiFreezeDataTableEntryPtr;
        // ...
    } FnVm;
    UnkSz1 vUnk1;
    struct _DetectionRoutine {
        UINT64 Type;
        PVOID TargetRoutine;
        DWORD DetectionSize;
        DWORD Hash;
        UINT64 Desire0, Desire1, Desire2;
    } *DetectionRoutines;
    UnkSz2 vUnk2;
    UnkSz3 MethodSpecific;
    UnkSz4 vUnk3;
    // ======== After 0xAE8 Size =========
    struct _PgCodeSection {
        UnkSz5 vUnk4;
        UCHAR InitKDbgSection[sizeof(INITKDBG)];
        UnkSz6 vUnk5;
    } PgCodeSection;
}
```

- `CmpAppendDllSectionCode`: `CmpAppendDllSection` 함수의 바이트코드를 저장하는 곳입니다. 후에 컨텍스트 진입점으로 활용됩니다. (섹션 3.2.5)
- `FnVm`: `KiInitializePatchGuard` 함수에서 사용하는 NT 함수들의 포인터를 저장합니다. 이는 분석가들을 방해하기 위한 것(난독화 목적)으로 보입니다. 이것이 선언된 이후 거의 모든 함수는 간접 호출로 호출됩니다.
- `DetectionRoutines`: 검사하는 함수들을 담은 동적 배열입니다.
- `MethodSpecific`: 각 `method` 파라미터(2번째)에 대해 개별적으로 저장되는 파라미터입니다.
- `PgCodeSection`: PG는 중단점, 후킹, 루틴 발견 등 Flow Instrumentation을 막기 위해 코드 영역을 컨텍스트에 따로 저장합니다. 할당할 때 크기가 `0x100000` 추가된 이유가 이 때문입니다.
- `InitKDbgSection`: `ntoskrnl.exe`의 `INITKDBG` 섹션을 통째로 갖다 둔 곳입니다.

### Random Padding

PG Context를 포함한 PG와 연관된 구조체는 랜덤한 수의 랜덤한 바이트 (총 2047, `0x7FF`개)를 구조체의 앞, 뒤에 추가합니다. 이로 인해 메모리에서 PG 관련 구조체 찾기가 더욱 어려워집니다.

<img src="/pg_bypass/PGStructOverview.png">

하지만 이는 좋은 점 또한 있는데, 바로 16 바이트 정렬이 깨진다는 것입니다. 다른 NT 구조체의 경우 대부분 16 바이트로 정렬되어 있는 반면에 PG 구조체 포인터의 경우 16 바이트 정렬이 안 되어 있어, 이것이 PG 구조체인지 쉽게 확인이 가능해집니다.

### Critical Routines

PG가 특별히 중요하게 여기는 루틴들이 있는데, 이것을 저는 Critical Routine이라고 부르기로 결정하였습니다. 목록은 다음과 같습니다.

<img src="/pg_bypass/CriticalRoutines.png">

가장 첫 번째 원소는 현재 `NULL`이지만, 런타임에 `HaliHaltSystem`으로 초기화 됩니다.

이 Critical Routine들은 변형될 시 안전한 시스템 종료에 영향이 가는 함수들이기 때문에, 따로 PTE까지 저장하여 둡니다. 이는 나중에 검사에 걸려 BSOD를 유발해야 할 시 복구됩니다.

### Encryption

PG는 자신을 암호화하여 저장합니다. 이는 초기화가 거의 완료되었을 때 실행하며, PG를 런칭하기 전에 암호화를 완료합니다. 기본적으로 XOR 키로 암호화되며, 이중 암호화 구조를 띄고 있습니다.

첫 번째 암호화는 `CmpAppendDllSection`, `KiDpcDispatch` 에서 해제되는 암호화입니다. 이 암호화는 코드 섹션을 포함한 전 구간을 암호화하며 PG Context 자체를 메모리에서 못 찾게 하는 역할을 합니다.

두 번째 암호화는 각종 검사 루틴 (대표적으로 `FsRtlMdlReadCompleteDevEx`)에서 해제되는 데이터 암호화입니다. 이 암호화는 설령 분석자가 코드 섹션에 어찌저찌 접근을 하였다 하더라도 데이터에 접근하지 못하게 막는 역할을 합니다.

### Entry Point

모든 PG Context의 진입점은 세 함수 중 하나로 직결됩니다: `CmpAppendDllSection`, `KiDpcDispatch`, `KiTimerDispatch`. `KiInitializePatchGuard`의 1번째 파라미터 dpcIndex에 따라
`PgDpcRoutines[dpcIndex]`로 pgDpc가 결정됩니다. `PgDpcRoutines` 배열은 다음과 같습니다.

```c++
PVOID PgDpcRoutines[13] = {
    CmpEnableLazyFlushDpcRoutine,
    CmpLazyFlushDpcRoutine,
    ExpTimeRefreshDpcRoutine,
    ExpTimeZoneDpcRoutine,
    ExpCenturyDpcRoutine,
    ExpTimerDpcRoutine,
    IopTimerDispatch,
    IopIrpStackProfilerDpcRoutine,
    KiBalanceSetManagerDeferredRoutine,
    PopThermalZoneDpc,
    KiDpcDispatch, // Or KiTimerDispatch
    KiDpcDispatch, // Or KiTimerDispatch
    KiDpcDispatch // Or KiTimerDispatch
};
```

이들은 모두 PG의 초기 진입점이 되는 DPC 루틴들으로, 여타 다른 정상적인 DPC들과 같이 `DeferredContext`가 존재합니다.

이 중 처음 열 개의 루틴에 대해 자세히 살펴보면, 모두 `KiCustomAccessRoutineN`과 관련이 있다는 것을 확인할 수 있습니다. 이 DPC 루틴들은 본래 정상적인 역할을 수행하다가, `DeferredContext`로 Canonical하지 않은 주소가 전달되면 `KiCustomAccessRoutineN`을 호출합니다. 이 `KiCustomAccessRoutineN`들은 내부적으로 `KiCustomRecurseRoutineN`을 각각 호출합니다.

이제 `KiCustomRecurseRoutineN`을 계속해서 다음 함수를 호출합니다. `...Routine0` 은 `...Routine1` 을 호출하고, `...Routine1` 은 `...Routine2`, `...Routine9`는 다시 `...Routine0` 으로 다시 돌아오는 식입니다.

이처럼 재귀하며 돌다가 `DeferredContext` 값을 포인터로 역참조를 시도합니다. 이때 `DeferredContext`는 유효한 주소가 아니기 때문에 예외가 발생하고, 숨겨진 핸들러로 이동하게 되어 결국 `CmpAppendDllSection`을 호출하게 됩니다.

<img src="/pg_bypass/RussianRoulettesRoutines.png">

`CmpAppendDllSection` 함수는 이제 자신의 컨텍스트를 복호화한 뒤 검사 루틴으로 들어갑니다. `KiDpcDispatch` 함수는 저러한 복잡한 과정 없이 직접적으로 자신의 컨텍스트를 복호화하고 검사 루틴을 호출합니다.

`KiTimerDispatch` 함수를 엔트리 포인트로 가진 PG Context는 `KiDpcDispatch`와 마찬가지로 복잡한 과정 없이 직접적으로 호출하지만, 자신의 코드를 복호화하는 첫번째 복호화 과정을 수행하지 않습니다.

### Detection Structure

PG는 NT 함수의 무결성을 검사할 때 특정한 구조체를 이용합니다. 이는 전에 언급한 구조체 `_DetectionRoutine`입니다.

```c++
struct _DetectionRoutine {
    UINT64 Type;
    PVOID TargetRoutine;
    DWORD DetectionSize;
    DWORD Hash;
    UINT64 Desire0, Desire1, Desire2;
} *DetectionRoutines;
```

- `Type`: 버그 체크의 유형입니다. `CRITICAL_STRUCTURE_CORRUPTION KeBugCheckEx`의 네 번째 파라미터와 일치합니다.  
- `TargetRoutine`: 타겟 함수의 주소입니다. 반드시 함수일 필요는 없으며, 특정 커널 오브젝트일 수도 있습니다. (예: IDT, SSDT 등)  
- `DetectionSize`: 검사할 크기입니다.  
- `Hash`: 검사의 결과가 정상적일 경우 도출되는 해시 값입니다.  
- `Desire0, Desire1, Desire2`: `Type`에 따른 특정 동작을 제어합니다.  
  - 예: IDT를 검사할 경우 일부러 Exception을 발생시키거나,  
  - `EPROCESS` 구조체를 검사할 때는 `Flink`와 `Blink`를 검사하는 것을 정의합니다.  
  - 검사 루틴에서 이 필드들을 이용해 특정 동작을 수행합니다.  

### VM Detection

저는 테스트 중 상당수를 VM에서 진행하였는데, VM에서 PG의 동작이 변경되는 것을 포착하였습니다. 이는 Windows 11에 들어서 새로 생긴 기능으로, 하이퍼바이저가 존재하는 환경에서 PG의 움직임이 크게 달라집니다. 대표적으로 `KeExitRetpoline` 함수를 통해 특정 VM 환경에서 검사를 아예 스킵하는 행위가 있습니다.  

이 외에도 `INITKDBG` 섹션 페이지에 `KiErrata`로 시작하는 함수들은 모두 하이퍼바이저 모듈을 감지하는 비정규적 방법을 사용하고 있습니다.  

이에 대해 자세히 파헤쳐 보지 않아 얼마나 많은 하이퍼바이저 감지 기술이 존재하는지 알 수 없지만, 분명한 것은 VM 환경은 PG를 분석하기에 좋지 않은 환경이라는 것입니다.

### Global PG Context

`method`가 7인 경우 `MaxDataSize` 전역 변수에 PG Context가 초기화됩니다. 이때 `MaxDataSize`라는 심볼이 직접적으로 확인되지는 않지만, IDA로 디컴파일할 경우 자동으로 `MaxDataSize`로 해석됩니다.  

전역 변수에 저장되는 컨텍스트는 항상 코드를 복호화하는 1단계 암호화가 빠져 있으며, 그에 따라 `KiTimerDispatch` 함수를 진입점으로 항상 이용합니다. `MaxDataSize`는 추후 `KiSwInterruptDispatch` 함수에서 검사할 때 이용되며, TV 콜백에 전달되는 `PgTVCallbackDeferredRoutine`에서도 사용됩니다.

### Code Section

PG는 관련된 PG 함수, 예를 들어 `FsRtlMdlReadCompleteDevEx`, `CmpAppendDllSection`와 같은 함수를 따로 컨텍스트 코드 영역에 저장하고, 해당 PG 함수를 수행할 때 `ntoskrnl.exe`가 아닌 PG Context에서 실행합니다. 이는 분석가들이 PG 함수에 후킹이나 BP를 거는 것을 방지하기 위함입니다.

### Applies

이렇게 초기화된 PG Context가 실제로 적용되는 부분을 살펴보겠습니다. `method` 값에 따라 PG가 실제로 적용되는 양상이 달라집니다.

#### Method 0

`method`가 0일 경우 `dpcIndex`에 의해 결정된 `pgDpc`를 DPC로 사용하는 타이머를 초기화한 뒤, `KeSetCoalescableTimer` 함수로 타이머를 큐합니다.  

이때 주목해야 할 점은 큐 되는 `KTIMER` 오브젝트의 `Period` 멤버가 0이고, `DueTime` 멤버에 약 2분 정도의 값이 들어 있다는 것입니다.  
“PG의 경우 계속해서 검사를 수행해야 할 텐데 왜 `Period` 값이 아닌 `DueTime`을 쓰는가?” 하는 의문이 들 수 있습니다.  

검사 루틴을 확인해 보면 모두 검사를 마친 뒤 다시 `KeSetCoalescableTimer` 함수로 타이머를 큐하는 것을 볼 수 있습니다. 즉, 추가적인 보안을 위해 `DueTime`을 선택하고 검사가 끝날 때마다 새로운 타이머를 넣는 방식을 택한 것입니다.

#### Method 1

`method`가 1일 경우 결정된 `pgDpc`를 바탕으로 `KPRCB.AcpiReserved` 멤버에 해당 DPC를 삽입합니다. 해당 DPC가 어떻게 호출되는지는 명확하지 않지만, 결과적으로 호출되는 것을 확인했습니다.

#### Method 2

`method`가 2일 경우 `method 1`과 거의 유사하지만 `KPRCB.HalReserved[7]` 멤버에 해당 DPC를 삽입합니다. `method 1`과 마찬가지로 최종적으로 호출되는 것을 확인했습니다.  

#### Method 3

`method`가 3일 경우 `pgDpc`는 사용되지 않습니다. 대신 `fiberParam` 내의 `PsCreateSystemThreadPtr` 함수 포인터로 스레드를 만든 뒤 `PsCreateSystemThreadDeferredStub` 함수를 삽입합니다.  

이 외에도 6개의 다른 함수가 삽입되는데, 이 함수들은 호출되지 않았습니다.  

해당 함수 내에는 검사 간격을 확보하기 위해 다음 세 개의 함수가 사용됩니다:  
- `KeDelayExecutionThread`  
- `KeWaitForSingleObject`  
- `KeWaitForMultipleObjects`  

언뜻 보면 `fiberParam` 내의 함수 포인터가 필요한 것 같지만, 이것이 진정 `fiberParam`을 필요로 하는지는 의문입니다. 실제로는 `fiberParam`이 없는 경우에도 `PsCreateSystemThread` 함수가 호출되는 것이 포착되었고, 정상적으로 PG Context가 초기화되는 모습 또한 확인되었습니다. 아마 다른 경로로 호출되는 것으로 보입니다.  

#### Method 4

`method`가 4일 경우 이미 존재하는 랜덤한 시스템 스레드에 `KeInsertQueueApc` 함수를 이용해 약 2분 간격의 APC를 주입합니다. 이 APC는 `method 3`과 마찬가지로 프로시저 실행에 지연을 발생시키는 함수를 사용하여 검사 간격을 확보합니다.  

#### Method 5

`method`가 5인 경우, `fiberParam`이 `NULL`일 경우에는 `method`가 0으로 폴백됩니다. 따라서 실질적으로 `method`가 5로 동작할 확률은 매우 낮습니다.  

이 방식은 `fiberParam`의 멤버 `KiBalanceSetManagerPeriodicDpcPtr`를 이용해 `KiBalanceSetManagerPeriodicDpc` 전역 변수를 후킹합니다. 내부적으로 카운터를 두어 정상적인 루틴을 수행하다가 카운터가 0이 되면 `pgDpc`를 큐하는 방식으로 동작합니다. 카운터는 초기에 랜덤한 값으로 초기화되며, 0이 될 때마다 `pgDpc`가 호출되고 다시 랜덤 값으로 재설정됩니다.  

<img src="/pg_bypass/method5.png">

#### Method 7

`method`가 6인 경우는 관찰되지 않았으며, 그 원인에 대해서는 아직 정확하게 규명되지 않았습니다.  

`method`가 7인 경우 `KiInitializePatchGuard`에서는 아무런 행위를 하지 않습니다. 다만 `KiInterruptThunk` 혹은 `KiMachineCheckControl` 함수로 DPC를 초기화하는 모습까지는 확인되었으나, 해당 DPC를 큐하지 않고 `KiInitializePatchGuard`가 종료됩니다.  

실제로 `KiMachineCheckControl`, `KiInterruptThunk`를 배제하고 우회 코드를 작성해도 전혀 문제 없이 우회가 정상적으로 수행되는 것을 확인할 수 있었습니다.  

`method 7`은 시스템 부팅 시 적어도 한 번은 호출됩니다. 이는 `KiFilterFiberContext`에서 마지막으로 항상 고정된 파라미터를 두기 때문입니다. `method 7`은 글로벌 PG Context 포인터인 `MaxDataSize` 전역 변수를 초기화하는데, 이 전역 변수가 항상 초기화되는 것을 `KiSwInterruptDispatch`에서 기대하기 때문에 그런 것으로 보입니다.  

자세한 정보는 섹션 3.3.4 를 참고하세요.

## Detections

이제 PG가 어떤 식으로 시스템 손상을 감지하는가를 알아보겠습니다.  
`_DetectionRoutine` 구조체(섹션 3.2.6)의 `Type`에 따라 검사 방법이 천차만별입니다. 또한 `DesireN` 멤버를 이용해 각 `Type`에 대해 개별적으로 행해지는 동작들을 정의할 수 있습니다.

### BugCheck Parameter

다음과 같은 인라인 블록들이 다수 존재합니다.

```c++
*(_QWORD *)(v2 + 2336) = v2 - 0x5C5FC0A76E374B18LL;
*(_QWORD *)(v2 + 2344) = (char *)v38 - 0x4C48B4211BBACBEBLL;
v68 = *v38;
*(_QWORD *)(v2 + 2360) = v67;
v69 = *(_DWORD *)(v2 + 2520);
*(_QWORD *)(v2 + 2352) = v68;
*(_DWORD *)(v2 + 2328) = 1;
if ( (v69 & 0x20000000) == 0 && (*(_DWORD *)(v2 + 2524) & 0x200000) != 0 && (v69 & 1) != 0 )
{
    v70 = *(unsigned int *)(v2 + 2676);
    v71 = *(_QWORD *)(v2 + 2104);
    v72 = *(_QWORD *)(v2 + 2680);
    v73 = (_QWORD *)(v70 + v2);
    v74 = v70 + v2 + 8 * ((unsigned __int64)(unsigned int)(*(_DWORD *)(v2 + 2052) - v70) >> 3);
    while ( v73 != (_QWORD *)v74 )
    {
        *v73 ^= v72;
        v72 = ((v71 ^ *v73++) + __ROR8__(v72, v72 & 0x3F)) ^ 0xEFA;
    }
    *(_DWORD *)(v2 + 2524) &= ~0x200000u;
    if ( v72 != *(_QWORD *)(v2 + 2688) )
    {
        v75 = *(_DWORD *)(v2 + 2052);
        v76 = *(_QWORD *)(v2 + 1416);
        *(_QWORD *)v76 = v2;
        *(_DWORD *)(v76 + 16) = v75;
        if ( !*(_DWORD *)(v2 + 2328) )
            *(_QWORD *)(*(_QWORD *)(v2 + 1416) + 24LL) = v72 ^ *(_QWORD *)(v2 + 2688);
        sub_140BCC384(a1: v2, a2: 0LL, a3: v72, a4: 256LL);
    }
}
```

이는 검사를 통과하지 못했을 때 호출되는 공통된 루틴으로, `KeBugCheckEx`에 들어가는 버그 체크 파라미터를 정의합니다. 즉, 예약된 파라미터의 정체를 알 수 있습니다.

- `Parameter1`: 검사를 수행한 PG Context 주소  
- `Parameter2`: 검사를 통과하지 못한 `_DetectionRoutine` 구조체의 주소  
- `Parameter3`: 손상된 데이터의 주소  
- `Parameter4`: 손상된 데이터의 Type  

`Parameter1`, `Parameter2`는 각각 상수 `0x5C5FC0A76E374B18`과 `0x4C48B4211BBACBEB`와 뺄셈을 수행하여, 분석가들이 접근하는 것을 일차적으로 저지합니다.  

`Parameter4`에 관련하여, 공식적으로 문서화되지 않은 `265`와 같은 Type이 존재하는데, 이는 대부분 PG Context의 손상과 관련된 경우입니다.  

해당 인라인 블록은 검사 루틴을 정적 분석할 때 매우 유용합니다. 어느 부분에서 검사가 실패했는지를 쉽게 파악할 수 있기 때문입니다.

### Main: FsRtlMdlReadCompleteDevEx

모든 `Type`에 대한 검사를 한꺼번에 진행하는 거대한 함수입니다. 함수는 우선 데이터 부분을 복호화한 뒤, Microsoft BugCheck 문서에 정의된 모든 `Type`에 대해 검사를 수행합니다. 마지막으로, 정리 루틴을 수행하는데, 여기서는 만료된 타이머를 다시 큐하는 등의 작업을 수행합니다.  

검사를 통과하지 못했을 경우 `CriticalRoutines`에 대한 PTE를 복구하고 `SdbpCheckDll` 함수를 호출합니다. 이 함수는 `KeBugCheckEx`를 호출하는 래퍼 함수이지만, 호출하기 전에 스택을 정리합니다. 이는 메모리 덤프를 분석하려는 분석가들을 방해하기 위한 방법입니다.

<img src="/pg_bypass/FsRtl.png">

#### Nb: PgDetectionN

이 외에도 심볼이 존재하지 않는 검사 함수가 몇 가지 있습니다. 저는 이들을 간단하게 `PgDetectionN`이라고 이름 지었습니다. 이 함수들은 모두 `FsRtlMdlReadCompleteDevEx`와 거의 동일한 동작을 하지만, 검사 `Type` 범위가 더 작다는 차이가 있습니다.  

또한 이들은 VM이 실행 중일 경우에만 동작하는 것이 확인되었으며, Bare-Metal 환경에서는 동작하는 것을 포착하지 못했습니다.  

이들을 자세히 분석하지 않은 이유는 VM에서만 동작하기 때문입니다. 우리의 목표는 Bare-Metal 환경에서 우회를 구현하는 것이지, VM 환경에서 구현하는 것이 아니기 때문입니다.

### Sub: PgTVCallbackDeferredRoutine

`PgTVCallbackDeferredRoutine`는 TV 콜백에 파라미터로 전달되는 함수로, `FsRtlMdlReadCompleteDevEx`와 동일한 역할을 수행하지만 마지막에 타이머, 스레드를 다시 큐하는 작업이 없습니다. 따라서 이 콜백이 어떻게 재호출되는지를 확인할 필요가 있습니다.  

해당 함수는 **Method 7**에서 초기화된 `MaxDataSize` PG 글로벌 컨텍스트를 사용합니다.  

### Sub: KiSwInterruptDispatch

최근에 추가된 검사 루틴으로 보입니다. 이 함수는 `KiSwInterrupt` IDT 함수에서 호출되는 루틴입니다. 이름과는 달리, 실제로는 PG 검사를 수행합니다.  

이 또한 마찬가지로 **Method 7**에서 초기화된 `MaxDataSize`를 사용합니다.  

### Sub: CcBcbProfiler

이 함수는 PG Context와는 독립적으로 동작하는 루틴으로, 엄밀히 말하면 `KiInitializePatchGuard`와 직접적인 관련은 없습니다.  

`CcInitializeBcbProfiler`에서 `KeSetCoalescableTimer` 함수를 이용해 타이머를 초기화하며, 약 2분마다 랜덤한 NT 커널의 한 함수를 조사합니다. 또한 쌍둥이 함수 `CcBcbProfiler2`(원래 심볼 없음)가 존재하며, 이 두 함수는 서로 협력하여 검사 루틴이 끝날 때마다 다시 `KeSetCoalescableTimer`로 큐되어 재실행됩니다.  

다른 점은, BSOD를 `SdbpCheckDll`이 아닌 `CcAdjustBcbDepth`를 통해 호출한다는 것입니다. 두 함수의 역할은 동일합니다.  

### Sub: KiMcaDeferredRecoveryService

이 외에도 독립적으로 운행되는 루틴이 여러 가지 있습니다. (예: `PspProcessDelete`, `KiInitializeUserApc`)  
이 함수들은 독립적인 검사 루틴을 수행한다기보다는, 원래의 정상적인 역할을 수행하기 직전에 가볍게 검사를 수행하는 성격을 가집니다.  

이 함수들은 공통적으로 BSOD를 발생시킬 때 `KiMcaDeferredRecoveryService`를 호출합니다.  
`KiMcaDeferredRecoveryService`는 `SdbpCheckDll`, `CcAdjustBcbDepth`와 동일하게 스택을 정리한 후 `KeBugCheckEx`를 호출하는 함수입니다.  

## Approach

앞서 살펴본 바와 같이, PG를 분석하기 위해서는 Breakpoint, 후킹, Exception 캐칭 등의 방법을 사용할 수 없으며, PG 루틴 자체에 포함된 안티 디버깅 모듈 때문에 디버깅을 통한 접근이 어렵습니다. 따라서 동적 분석을 원활하게 수행하기가 매우 힘듭니다.  

또한 난독화와 인라인화 기법으로 인해 정적으로 PG를 분석하는 난이도 역시 크게 상승합니다. 이러한 제약 조건 속에서 PG를 분석하기 위해 저는 창의적이고 다양한 분석 기법을 활용하였습니다.  

섹션 3 에서 제시된 분석 결과는 지금부터 설명할 방법들을 통해 도출된 것입니다.

### Static Analysis

실제로 저는 전체 분석 시간 중 가장 많은 시간을 이 정적 분석에 소모하였습니다. 다행인 점은 PG 함수가 가상화되어 있지 않고 난독화만 적용되어 있기 때문에, 충분한 노력을 기울이면 코드 분석을 수행할 수 있다는 점입니다.  

분석을 진행하다 보면 인라인 처리된 것으로 추정되는 부분들을 다수 발견할 수 있습니다.

<img src="/pg_bypass/PoolAllocation.png">

예를 들어, `rdtsc` 명령으로 시작하여 분기가 많이 존재하는 인라인 블록은 풀 할당 시 풀 태그로 식별되지 않도록, 랜덤한 알파벳 네 글자를 생성하는 데 사용되는 인라인 어셈블리입니다.  

분석 과정에서는 이러한 인라인 블록들을 파악하고 빠르게 넘어가는 데 집중했습니다. 정적 분석을 수행할 때는 특별한 도구나 트릭을 사용하지 않고, 정직하게 시간을 투자하며 진행했습니다.  

그러나 역설적이게도, 이는 가장 성과가 적었던 방법이었습니다.

### Length Trick

앞서 언급했듯이 PG에는 난독화가 적용되어 있는데, 이러한 난독화 기법의 특성상 코드의 길이가 불필요하게 늘어나는 경향이 있습니다.  

이 점에 착안하여 `ntoskrnl.exe` 내의 함수들을 코드 길이가 긴 순서대로 정렬해 보니, 실제로 다수의 PG 루틴들이 그 안에 포함되어 있는 것을 확인할 수 있었습니다.

<img src="/pg_bypass/FunctionLengths.png">

### prefetchnta Trick

PG와 관련된 중요한 루틴들은 대개 `prefetchnta byte ptr [rax]` 명령을 포함합니다. 해당 옵코드를 왜 호출하는지는 정확히 알 수 없지만, 아마도 검사를 수행할 때 성능을 향상시키기 위한 목적이라고 추측할 수 있습니다.

<img src="/pg_bypass/prefetchnta.png">

따라서 PG를 분석할 때 시그니처 `0F 18 00`을 검색하면 효과적으로 루틴을 발췌하여 분석할 수 있습니다.  

### xref Tracing

NT 관련 함수들은 간접 호출을 거의 사용하지 않기 때문에, 특정 함수의 Cross Reference를 조사하면 해당 함수를 호출하는 부모 함수가 그대로 드러날 가능성이 큽니다.  

### Obfuscation makes sense!

특정 함수를 보았을 때 "이것이 PG 함수인가?"라는 질문에 답할 수 있게 해주는 것은 역설적이게도 **난독화 자체**입니다.
PG 함수들은 난독화를 위해 일반적으로 잘 사용하지 않는 `rdtsc`, `rol`, `ror` 등의 어셈블리어 명령어를 자주 사용하기 때문에, 이를 단서로 삼으면 해당 함수가 PG 함수인지 더 쉽게 파악할 수 있습니다.

### PG Constants

BSOD를 유발할 때 사용되는 두 가지 상수가 존재합니다:  
- `0x5C5FC0A76E374B18`  
- `0x4C48B4211BBACBEB`  

이 상수들은 버그 체크 코드 `0x109`와는 달리 암호화되지 않은 평문이기 때문에 시그니처 검색에 활용할 수 있습니다.  

이 중 두 번째 상수(`0x4C48B4211BBACBEB`)는 `NULL`인 경우도 존재하므로, 저는 안정적인 검색을 위해 첫 번째 상수(`0x5C5FC0A76E374B18`)를 사용하였습니다.

## Dynamic Analysis

PG는 동적 분석이 매우 어려운 프로그램에 속합니다. Windows에 특화된 디버거인 [WinDbg](https://learn.microsoft.com/ko-kr/windows-hardware/drivers/debugger/)는 사용할 수 없습니다. WinDbg를 사용하려면 Windows가 디버그 모드로 실행되어야 하는데, PG는 디버그 모드에서는 아예 활성화되지 않기 때문입니다.  

혹자는 "`Kd` 관련 플래그를 원상태로 복구하면 되지 않는가?"라고 생각할 수 있습니다. 하지만 디버깅 관련 플래그가 정확히 몇 개인지도 파악하기 어렵고, `KdDisableDebugger`, `cli`-`sti` 등을 포함한 다양한 안티 디버깅 루틴이 존재하기 때문에 이를 모두 우회하거나 복구하기에는 소모가 지나치게 큽니다.

### VM

하이퍼바이저(VM)를 외곽 디버거로 활용하면, 게스트 OS 입장에서는 인터럽트나 디버그 레지스터와 같은 흔적이 전혀 남지 않은 상태에서 완전히 **Out-of-band 디버깅**이 가능합니다.  

핵심은 **VM-exit 트랩을 디버거 이벤트로 변환하는 것**입니다. 예를 들어 인텔 VT-x의 경우, 게스트 OS가 Debug Register를 조작하는 연산은 하이퍼바이저에 먼저 전달되어 VM-exit이 트리거됩니다. 이를 하이재킹하여 디버거 이벤트로 변환하는 것이 구체적인 방법입니다.

<img src="/pg_bypass/VMDebugging.png">

[VMware](https://www.vmware.com/)는 해당 기능에 대한 지원이 있습니다. VM의 설정 파일(\texttt{.vmx})에 다음과 같은 항목을 추가하면 됩니다.

```
debugStub.listen.guest64 = "TRUE"
monitor.debugOnStartGuest64 = "TRUE"
debugStub.port.guest64 = "1234"
```

이렇게 작성하고 실행하면 `debugStub.port.guest64` 포트에 [gdb](https://sourceware.org/gdb/)가 열리게 되며, [IDA](https://hex-rays.com/) 디버그 프론트엔드에서 해당 VM에 **Remote GDB Debugger**로 연결하여 정상적으로 VM을 디버깅할 수 있습니다.  

VM을 사용할 경우 앞서 분석했듯이 Control Flow가 달라져 PG가 정상적으로 동작하지는 않지만, 초기화 루틴(`KiInitializePatchGuard`)을 분석하기에는 매우 유리합니다.  

특히 초기화 단계에서 호출되는 함수들, 그리고 간접 호출을 파악하는 데 용이하며, PG Context의 특정 부분에 R/W Trace를 걸어 어떤 값이 어떤 시점에 복사되는지를 직관적으로 확인할 수 있습니다. 실제로 타이머와 관련된 대부분의 루틴 동작은 이 방법을 통해 유추해낼 수 있었습니다.  

단점으로는, 디버깅이 `KiSystemStartup` 함수가 아닌 **EFI 로딩 단계**에서 시작되기 때문에 정확한 진입점을 찾기 어렵다는 점이 있습니다. 또한 이 시점에서는 `ntoskrnl.exe`가 메모리에 할당되는 주소와 같은 필수적인 정보들을 확인할 수 없습니다.  

저희의 목표는 NT 커널이 초기화될 때 함께 초기화되는 PG를 분석하는 것이므로, 이 문제는 상당히 치명적입니다.  

이를 해결하기 위해 저는 `KUSER_SHARED_DATA` 구조체에 주목했습니다. `KUSER_SHARED_DATA`는 시스템의 고정된 주소(`FFFF'F780'0000'0000`)를 사용하며, 시스템 초기화에 활용되는 함수입니다. 실제로 `KiInitializeKernel` 함수에서 `KUSER_SHARED_DATA+280`을 참조하는 것을 확인할 수 있었습니다.

<img src="/pg_bypass/KiInitializeKernel.png">

이제 방법은 간단합니다. 우리는 `FFFF'F780'0000'0280`에 **Hardware R/W Breakpoint**를 설정할 수 있으며, 실제 실행해 보면 `ntoskrnl.exe`가 메모리에 할당된 직후에 해당 BP가 트리거되는 것을 확인할 수 있습니다.  

하지만 여기까지의 모든 과정을 매번 디버깅할 때마다 수동으로 수행하는 것은 비효율적입니다. 따라서 자동화를 위해 **IDAPython Script**를 작성하였으며, 코드는 Git 저장소의 Research 폴더에서 확인할 수 있습니다: [GitHub - kurasagi/research](http://github.com/NeoMaster831/kurasagi/tree/7d8c832dea17e0b35971ddde0c6afa0bba315dd9/research)  

이 과정을 마치면 초기화 루틴을 보다 정밀하게 분석할 수 있습니다. 또한 OS가 실행 중일 때 언제든지 OS를 **Suspend**하여 분석하는 것도 가능합니다.

### Barricade

이 방법은 PG 분석에 있어 매우 치명적인 역할을 했습니다. 이는 정형화된 기법이라기보다는, PG의 특성을 면밀히 분석하여 도출한 방법론입니다.  

PG Context는 자신을 암호화하여 저장하며, `CmpAppendDllSection` 혹은 `KiDpcDispatch`를 진입점으로 사용하는 컨텍스트는 해당 함수들을 통해 진입하면서 스스로를 복호화합니다. 실행 가능한 어셈블리 코드이므로 실행 권한(X)이 필요하다는 것은 자명합니다. 또한 해당 페이지 안의 컨텍스트를 복호화해야 하므로 읽기/쓰기 권한(R/W) 역시 필요합니다.  

따라서 두 함수를 진입점으로 하는 **패치 가드의 진입점은 항상 RWX 권한을 가지고 있습니다.**

<img src="/pg_bypass/CmpAppendDllSectionRWX.png">

여기서 “PG Context 진입점은 항상 RWX인가?”라는 추론을 세울 수 있습니다. 실제로 `KiTimerDispatch` 함수를 포함해 세 함수의 페이지 PTE를 확인한 결과, 모두 RWX 권한을 가지고 있음이 밝혀졌습니다. 특히 `KiTimerDispatch` 함수는 RWX 권한이 없어도 되는데도 RWX였다는 점이 매우 놀라웠습니다.  

이 사실이 중요한 이유가 있습니다. 마이크로소프트의 드라이버 서명 정책을 살펴보면, **서명받은 드라이버는 RWX 섹션이 단 1바이트라도 존재해서는 안 됩니다.**  
[참고 문서: Driver compatibility with Device Guard](https://learn.microsoft.com/en-us/windows-hardware/test/hlk/testref/driver-compatibility-with-device-guard)  

또한 Inf2Cat은 `NonPagedPool`(=`NonPagedPoolExecute`) 풀이 존재할 경우 테스트를 통과하지 못하도록 설정되어 있습니다.

<img src="/pg_bypass/DriverEnforcement.png">

즉, 정상적인 Windows 11 시스템에서는 RWX 섹션을 가진 드라이버 코드가 존재하지 않아야 합니다. 여기서 **RWX 섹션을 가진 모듈은 PG뿐이다**라는 아이디어를 도출할 수 있습니다. (`ntoskrnl.exe`의 일부 모듈도 RWX 섹션을 사용하지만, 그 수는 매우 적습니다.) 이는 나중에 구현 과정에서 오버헤드와 오류가 적은 이유와도 연결됩니다.  

따라서 모든 페이지를 순회하면서 RWX 섹션 중 X 권한을 제거합니다. 이렇게 하면 PG 진입점에서 처음 코드를 실행하려 할 때 X 권한이 없기 때문에 **페이지 폴트(`#PF`) 예외가 발생**하게 됩니다.  

`#PF`를 담당하는 함수는 IDT의 `KiPageFault` 함수이며, 이 중에서도 실행 권한(X)이 없어 발생한 경우에는 `MmAccessFault` 함수로 리다이렉트됩니다.  

따라서 `MmAccessFault` 함수를 후킹하여 PG에 대한 예외만 추가적으로 처리할 수 있습니다. 저는 분석 과정에서 의도적으로 스택 정보를 포함시켜 BSOD를 발생시키는 방법을 사용하여 분석을 더 수월하게 만들었습니다.  

PG 분석이 어려운 이유는 크게 두 가지입니다.  
1. 컨텍스트가 암호화되어 있어 PG 루틴이 실행 중이 아닐 때는 컨텍스트를 발견하기 어렵다는 점.  
2. 루틴이 어디서, 어떻게 호출되는지 파악하기 어렵다는 점.  

그러나 이 방법은 두 가지 문제를 동시에 해결할 수 있는 강력한 기법입니다.  

저는 **"PG 루틴이 진입하기도 전에 `#PF`를 발생시켜 막는다"**라는 아이디어가 마치 진입을 차단하는 바리케이드(Barricade)와 닮아 있다고 생각하여, 이 방법을 **Barricade**라고 명명하였습니다.

<img src="/pg_bypass/BarricadeDebug.png">

이처럼 간단한 컴퓨터 과학적 증명을 통해 도출해낼 수 있는 이 방법론은 추후 PG 우회에서도 용이하게 쓰입니다.

# Bypass

저는 어떻게 우회를 해야 하는가에 대해 고찰하였습니다. 물론 가장 쉬운 방법 몇 가지를 꼽자면, 앞서 살펴본 것처럼 시스템 자체를 디버그 모드로 부팅하거나, VT-x를 이용해 EPT 후킹을 하여 손쉽게 우회할 수도 있습니다. 하지만 우리의 목표는 **베어메탈 환경에서 일반적인 시스템 런타임 중 PG를 우회하는 것**이므로, 이러한 방법론들은 사용할 수 없습니다.  

`KiInitializePatchGuard` 함수의 흐름을 직접 변경할 수 없다는 점이 중요한 제약으로 작용합니다. 그러나 앞서 언급했듯이 **적용 부분과 함수 호출 경위**를 명확히 이해한다면, 각 함수에 대해 개별적으로 우회 시나리오를 설계할 수 있습니다.

## Timers

현재까지 발견된 Timer 관련 적용 및 검사 루틴은 다음과 같습니다. 다음 모듈들을 모두 종료시켜야 합니다.

- `KiInitializePatchGuard` Method 0  
- `CcBcbProfiler` & `CcBcbProfiler2`  
- DPC: `KiInitializePatchGuard` Method 1  
- DPC: `KiInitializePatchGuard` Method 2  
- Hook: `KiInitializePatchGuard` Method 5  

method 1, 2 또한 결국에는 타이머와 연관이 있기 때문에 타이머에 포함하였습니다.  

타이머는 `PRCB.TimerTable`에 연결 리스트 형태로 저장되는 것이 일반적입니다. 따라서, 모든 프로세서를 순회하며 `PRCB.TimerTable`을 스캔하는 식으로 모든 레지스터된 타이머를 조사할 수 있습니다.  

문제는 타이머 엔트리 안의 DPC 루틴을 정의하는 `Dpc` 멤버는 `KiWaitAlways`, `KiWaitNever`와 암호화되어 저장되었다는 것입니다. 이는 `KeSetTimer` 함수의 초반에서 발견할 수 있었습니다.

```c++
v4 = (unsigned __int64)Timer ^ _byteswap_uint64(KiWaitAlways ^ (unsigned __int64)Dpc);
v42 = Dpc;
v5 = (unsigned int)KiWaitNever;
v6 = DueTime;
v7 = (_KDPC *)(KiWaitNever ^ __ROR8__(v4, KiWaitNever));
```

발견한 내용을 바탕으로 복호화하는 함수를 작성하면 다음과 같습니다.

```c++
PKDPC wsbp::Timer::GetDecryptedDpc(PKTIMER pTimer) {
    return (PKDPC)(*gl::RtVar::KiWaitAlwaysPtr ^ _byteswap_uint64(\
        (UINT64)pTimer ^ _rotl64(\
            (UINT64)pTimer->Dpc ^ *gl::RtVar::KiWaitNeverPtr, \
            (UCHAR)*gl::RtVar::KiWaitNeverPtr \
        )));
}
```

### KiInitializePatchGuard Method 0

Method 0에서 pgDpc로 전달되는 `DeferredContext` 값은 Canonical한 주소가 아니라고 언급한 적이 있습니다. (섹션 3.2.5)  

따라서, `DeferredContext`가 Canonical한 주소가 아닌 경우를 체크하여 발견될 경우 `Dpc` 자체를 `NULL`로 만들어 해결할 수 있습니다.  

MSDN에도 `Dpc`가 `NULL`이어도 된다고 나와 있습니다.  
[참고 문서: KeSetTimer function](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/wdm/nf-wdm-kesettimer)  

`DeferredRoutine`을 확인하지 않는 이유는 두 가지입니다.  
1. 원래 그 함수는 정상적인 역할을 하기 때문에 판별 근거가 되지 않습니다.  
2. `KiDpcDispatch` 및 `KiTimerDispatch` 함수는 앞서 언급했듯이 PG Context로 옮겨지기 때문에 판별하기 어렵습니다.  

### CcBcbProfiler Twins

이 경우는 더 간단합니다. `DeferredRoutine`을 근거로 확정할 수 있기 때문입니다.  
`DeferredRoutine`이 `CcBcbProfiler` 혹은 `CcBcbProfiler2`인지 확인하고, 해당될 경우 동일하게 `Dpc`를 `NULL`로 만들어주면 해결됩니다.  

### KiInitializePatchGuard Method 1, 2 DPCs

- Method 1: `KPRCB.AcpiReserved`  
- Method 2: `KPRCB.HalReserved[7]`  

이들은 본래 값이 `NULL`입니다. 따라서 `NULL`이 아닌 값이 목격될 경우 단순히 다시 `NULL`로 변경하면 됩니다.  

### KiInitializePatchGuard Method 5

`KiBalanceSetManagerPeriodicDpc` 변수를 원래 상태로 복구하면 됩니다. 원래 `DeferredRoutine`은 `KiBalanceSetManagerDeferredRoutine`입니다.  

### Implementation

전체 구현 코드는 Git 저장소에서 확인할 수 있습니다.  
[GitHub - kurasagi/Module/Timer.cpp](http://github.com/NeoMaster831/kurasagi/blob/master/kurasagi/Module/Timer.cpp)

## MaxDataSize References

Method 7에서 초기화되는 PG 글로벌 컨텍스트 `MaxDataSize`에 접근하고 사용하는 루틴은 다음과 같습니다.

- `KiSwInterruptDispatch`  
- `PgTVCallbackDeferredRoutine`  

이 경우 기본적으로 `MaxDataSize`를 `NULL`로 맞추는 것만으로 해결이 가능합니다. 하지만 문제가 있습니다. 대부분의 다른 루틴들은 `MaxDataSize`를 역참조하기 전에 `NULL`인지 확인하지만, `KiSwInterruptDispatch` 함수는 확인하지 않고 곧바로 역참조를 시도하기 때문에 BSOD를 유발합니다.  

따라서 이를 반영하여 **`KiSwInterruptDispatch` 함수 자체를 패치하여 검사를 스킵**할 수 있습니다. 이 방법은 PG가 이외에 활성화된 루틴이 없을 때 적용할 수 있습니다.  

### Implementation

전체 구현 코드는 Git 저장소에서 확인할 수 있습니다.  
[GitHub - kurasagi/Module/Context7.cpp](http://github.com/NeoMaster831/kurasagi/blob/master/kurasagi/Module/Context7.cpp)

## Miscellaneous

기타 루틴들은 다음과 같습니다.

- `KiMcaDeferredRecoveryService`  
- System Thread: `KiInitializePatchGuard` Method 3  
- APC: `KiInitializePatchGuard` Method 4  

### KiMcaDeferredRecoveryService

해당 함수를 패치하여 `KeBugCheckEx`가 동작하지 않도록 하면 됩니다.  
`SdbpCheckDll`과 `CcAdjustBcbDepth`를 호출하는 루틴은 `JUMPOUT` 명령을 통해 루틴이 실패했을 경우 트랩에 걸리도록 구성되어 있지만, `KiMcaDeferredRecoveryService`를 호출하는 루틴에는 이러한 장치가 없습니다. 이는 하나의 **취약점**으로 작용합니다.  

### System Thread & APC

쓰레드나 APC는 내부적으로 `KeDelayExecutionThread`, `KiWaitForSingleObject`, `KiWaitForMultipleObjects`를 호출하여 프로시저 실행을 지연시킵니다. (섹션 [Method 3](#sec:method_3) 참고)  

따라서 시스템 쓰레드를 순회하며 콜 스택을 조사하고, 리턴 주소에 이 세 함수 중 하나라도 있다면 `KeDelayExecutionThread`를 `DueTime`을 매우 큰 값으로 설정하여 호출함으로써 종료시킬 수 있습니다.  

### Implementation

전체 구현 코드는 Git 저장소에서 확인할 수 있습니다.  
[GitHub - kurasagi/Module/Misc.cpp](http://github.com/NeoMaster831/kurasagi/blob/master/kurasagi/Module/Misc.cpp)

## Barricade

바리케이드 방법은 바리케이드 디버깅 방법 (섹션 4.2.2)과 지금까지 제시된 검사 우회 방법들을 융합한 **총체적 검사 우회 기법**입니다.  

이 방법은 PG 진입점에서 실행 시 `#PF`를 발생시켜, 후킹된 `MmAccessFault` 함수를 통해 추가적인 조사를 수행하여 PG를 식별하는 방식입니다.  

문제는 `MmAccessFault` 함수가 여러 곳에서 호출된다는 점입니다. 따라서 발생한 예외가 PG 루틴에 의한 것인지 판별하기 원래라면 어려웠을 것이고, 오버헤드 또한 매우 컸을 것입니다. 그러나 우리가 증명한 사실에 따르면 **간단한 RWX 권한 체크만으로도 식별 정확도를 크게 높이고 오버헤드를 줄일 수 있음**이 밝혀졌습니다.

### Implementation

PG를 우회하는 전체 구현 코드는 Git 저장소에서 확인할 수 있습니다.  
[GitHub - kurasagi/Module/Barricade.cpp](http://github.com/NeoMaster831/kurasagi/blob/master/kurasagi/Module/Barricade.cpp)  

설명은 간단하지만 실제 구현은 상당히 어렵습니다. 코드를 직접 확인해 보실 것을 강력히 권장합니다.

# Mitigations

지금까지의 우회 기법을 고찰하며, 어디서 방어할 수 있을지 살펴보겠습니다.

## Barricade: PatchGuard is too OLD!

현재 가장 치명적인 방법은 Barricade입니다. 그러나 Barricade를 막을 방법은 명확히 떠오르지 않습니다. RWX 권한을 사용하지 않는다면 Signature Scan에 쉽게 뚫릴 수 있기 때문입니다.  

따라서 **발상의 전환**이 필요합니다. 커널에서 커널을 검사하지 말고, 더 높은 권한에서 검사를 수행하는 것입니다. 실제로 Microsoft도 이를 인지하고 있으며, HyperGuard(HG)로 대응을 시도하고 있습니다. 하지만 HyperGuard는 아직 불완전하고 기능도 제한적이라는 것이 알려져 있습니다.  

저는 HG에 상당한 가치를 둡니다. 앞으로 가상화 기술이 점차 대두됨에 따라 이를 사용하는 장치가 많아질 것이고, HG 사용을 강제할 수 있을 것이라고 믿습니다. 따라서 PG를 HG에 통합하는 것을 권장합니다.  

물론 지금 당장 급진적으로 통합을 요구하는 것은 아닙니다. 가상화 기술이 본격적으로 주류로 자리 잡기까지는 시간이 필요하다고 생각합니다. 그러나 이는 곧 **충분한 개발 시간을 확보할 수 있다**는 의미이기도 합니다. HG를 *Slow and Steady*하게 발전시켜 나간다면, Windows 보안을 한층 더 견고하게 만들 수 있을 것입니다.

## Something Else

이 외에도 제가 사용했던 보완 가능한 취약점들을 나열하면 다음과 같습니다.

- `KeBugCheckEx 0x109` 파라미터 제거 (및 인라인 블록 제거)  
- `prefetchnta` 명령 제거  
- `KiMcaDeferredRecoveryService` 후 트랩 추가  
- 간접 호출 추가  
- 필요하지 않은 난독화 제거  
- 고정된 상수 (예: `0x4C48B4211BBACBEB`) 제거, 버그 체크 코드처럼 `rol`, `ror`로 난독화  
- 성능에 문제되지 않는 선에서 가상화 기법 추가  
- `KiTimerDispatch` 섹션의 W 권한 제거  

# Conclusion

이상으로 PG 우회를 통해 살펴본 바와 같이, PatchGuard는 혁신적인 보호 기법으로서 대부분의 리버스 엔지니어들을 혼란에 빠뜨릴 만큼 강력하지만, 숙련된 공격자를 완전히 저지하기에는 여전히 한계가 존재합니다.  

따라서 Windows 커널 보안을 한 단계 더 끌어올리기 위해서는 PatchGuard의 잠재적 취약점을 보완하고, 이를 넘어서는 **하이퍼바이저 기반 다계층 방어 체계**를 도입 및 강화할 필요가 있습니다.  

본 고찰이 향후 커널 보안 연구와 실무적 방어 전략 수립에 작은 길잡이가 되기를 바랍니다. 감사합니다.

## Precautions with Demonstration

전체 코드는 Git 저장소에 있으며  
[GitHub - kurasagi](http://github.com/NeoMaster831/kurasagi)  

이 코드는 **PG 전체를 우회하는 코드**입니다. 즉, 드라이버를 로딩하면 PG는 종료됩니다.  

해당 코드는 **Microsoft Visual Studio 2022**로 빌드할 수 있으며, 이때 **WDK** 및 **Windows SDK**가 시스템 내에 설치되어 있어야 빌드가 가능합니다.

시연하실 경우 다음과 같은 주의사항을 반드시 숙지해 주세요.

1. 이 PG 우회 코드는 **Windows 11 24H2 Build 26100.4351**에서만 작동합니다.  
2. **코어 격리(Core Isolation)** 관련 옵션을 모두 비활성화해야 합니다.  
3. 저는 [kdmapper](https://github.com/TheCruZ/kdmapper/tree/master)를 사용하여 서명되지 않은 드라이버를 시스템에 로딩해 PG 우회를 실험하였습니다. 다만, 일반적인 방법으로 드라이버를 로딩해도 정상적으로 작동합니다.  
4. `VerifyPg`를 통해 NT 함수를 후킹하여 PG를 강제로 트리거할 수 있습니다. `kurasagi` 드라이버를 로딩할 경우 성공적으로 PG가 우회되어 BSOD가 발생하지 않는 것을 확인할 수 있습니다.