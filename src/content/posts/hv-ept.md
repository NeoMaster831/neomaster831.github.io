---
title: Hypervisor basics, pt. 2
published: 2024-06-18
description: 'Use EPT Hooking'
image: '/hv-ept/ept.png'
tags: [VT-x, Hypervisor, EPT]
category: 'Hypervisor'
draft: false
---

# Hypervisor Basics, pt 2

하이퍼바이저에 대한 설명 대충 끝마쳤으니 이를 이용해서 할 수 있는 아주 강력한 행위인 **EPT Hooking**을 소개한다.

Windows같은 경우 [DKOM](https://en.wikipedia.org/wiki/Direct_kernel_object_manipulation)을 허용하지 않는다. 보안 문제 때문이다. 하지만 HV를 이용한다면 DKOM을 할 수 있게 된다. 이는 커널 레벨에서 상주하는 타 다른 드라이버들과 구별되는 엄청난 어드밴티지이다.

## Paging

EPT를 알기 전에 일단 알아야 하는 선행 개념들이 많다. 바로 페이징(Paging)이다.

여러분들이 코딩할 때 나타나는, OS에서 보여주는 대부분의 주소들은 실제 값이 들어있는 '물리 주소'가 아닌 '가상 주소'이다. 이때 가상 주소를 읽어 값을 읽을 수 없으므로 OS는 가상 주소를 참조할 때 실제 주소로 매핑하는 과정을 수행하는데, 이를 페이징(Paging)이라 한다.

### Page Walking

자세히 내부적인 부분을 들여다 보면 OS는 Page Walking이라는 기법으로 가상 주소에서 물리 주소로 변환한다.

우선 현대 운영 체제에서 주소의 상위 16비트는 Canonical Bits로, `47`번째 비트가 0이라면 비트 `48:63`은 모두 0이여야 하고, 1이라면 모두 1이여야 한다.
유저 모드 주소가 `00007FFFFFFFFFFF` 와 같이 나타나고 커널 모드 주소가 `FFFF800000000000` 와 같이 나타나는 이유이다. `0123456789ABCDEF`나 `0000800000000000` 과 같은 주소는 `Canonical Address`가 아니므로 오류가 나게 된다.

OS는 4단계 주소 변환 기법을 사용한다. CR3 레지스터에는 PML4라는 List Entry가 존재하는데, PML4에는 비트 `39:47`에 대응하는 총 512개의 요소가 있다. 이에 대응하는 요소를 고르면 해당 PML4 Entry에 해당하는 PDPT List Entry가 새로 등장한다.

PDPT에는 비트 `30:38`에 대응하는 총 512개의 요소가 있다. 이에 대응하는 요소를 고르면 해당 PDPT Entry에 해당하는 PD List Entry가 새로 등장한다.

PD에는 비트 `21:29`에 대응하는 총 512개의 요소가 있다. 이에 대응하는 요소를 고르면 해당 PD Entry에 해당하는 PT List Entry가 새로 등장한다.

PT에는 비트 `12:20`에 대응하는 총 512개의 요소가 있다. 이에 대응하는 요소를 고르면 해당 PT Entry에 해당하는 Page가 등장한다.

즉, PML4 -> PDPT -> PD -> PT 순으로 트리 구조를 활용하여 관리한다.

Page는 메모리를 관리하는 기본 단위로, 기본적으로 `4KB` (`4096B`) 이다. 마지막 `0:11` 에 대응하는 비트들은 Offset으로, Page 처음 시작 부분에서 몇 바이트 떨어져 있는지 나타낸다.

OS는 이와 같이 메모리를 효율적으로 관리한다. Paging을 GVA to GPA (Guest Virtual Address to Guest Physical Address)라고도 한다.

## EPT

EPT, Extended Page Translation은 SLAT, Second Level Address Translation이라고도 불린다. 

페이징을 통해 얻은 GPA는 실제 Host의 주소가 아니다. 따라서 GPA를 HPA (Host Physical Address)로 변환하는 과정이 필요한데, 이것이 EPT이다.

EPT는 GPA를 한 번 더 9바이트로 나눠 변환한다. Page Walking이랑 과정이 거의 같고, PML4E -> PML3 (PDPTE) -> PML2 (PDE) -> PML1 (PTE) 순으로 실행된다.

정리해보면, (GVA) -> PML4 -> PDPT -> PD -> PT -> (GPA) -> PML4E -> PML3 -> PML2 -> PML1 -> (HPA) 순이다.

## EPT Hooking

EPT Hooking의 기본 아이디어는 GPA의 최종적인 PML1 Entry를 조작하여서 원하는 HPA로 리다이렉트 시키는 것이다.

PML1에는 몇 가지 제어할 수 있는 비트 (대표적으로 RWX 권한 설정)과 PFN이 존재한다. 우리는 이 PFN을 조작하여서 원하는 Page로 Redirect되게 만들면 되는 것이다.

이를 구현하기 위해 [Trampoline Hook](https://crazyhacker.tistory.com/74?category=1019167)을 사용한다.

### Implementation I

1.  후킹하고 싶은 함수 `hkFunc`, 후킹 당할 함수 `tgFunc`를 준비한다.
2.  `tgFunc`를 복사하여 `fkFunc`를 만든다.
3.  `fkFunc`의 처음 Opcode들을 `hkFunc`로 jmp하는 코드로 대체한다.
4.  `trFunc` 함수를 새로 만든다: `fkFunc`의 대체된 Opcode들을 실행하고, jmp 코드 다음으로 jmp하는 함수이다.
5.  `tgFunc`이 상주하는 PML1의 PFN을 `fkFunc`가 있는 곳으로 조작한다.
6. 후킹 성공! 원래 함수를 호출하고 싶은 경우 `trFunc` 함수를 실행하면 됨

### Problem

문제점이 존재한다. 타 프로세스에서 Execute로 접근하는 것이 아닌 Read/Write로 접근하는 경우 후킹 코드가 노출될 가능성이 존재한다.

PML1 구조체에는 아까 말했듯이 RWX 권한을 지정하는 플래그가 있고, 이를 위반할 경우 EPT Violation이란 명분으로 VM Exit이 발생하게 된다. 이를 이용해보자.

대체하려는 페이지를 두 개의 페이지로 나누자. 하나는 원본 `tgFunc`이고 하나는 `fkFunc`이다.

이때 `tgFunc`에 RW- 권한, `fkFunc`에 –X 권한을 부여한다.

기존에 Execute를 할 때는 `fkFunc`가 실행되지만 Read/Write를 할 경우 권한이 없어 EPT Violation이 발생하게 된다.

이때 VM Exit Handler에서 EPT Violation을 핸들링하여 이때에 한하여 `tgFunc`로 페이지를 교체한다.

다시, `tgFunc`에서 Execute 명령이 떨어졌을 때도 EPT Violation이 일어나게 되고 VM Exit Handler에서 다시 `fkFunc`로 교체한다.

이 과정을 통해 완벽히 *STEALTH* 한 후킹을 할 수 있다.

### Implementation II

1.  후킹하고 싶은 함수 `hkFunc`, 후킹 당할 함수 `tgFunc`를 준비한다.
2.  `tgFunc`를 복사하여 `fkFunc`를 만든다.
3.  `fkFunc`의 처음 Opcode들을 `hkFunc`로 jmp하는 코드로 대체한다.
4.  `trFunc` 함수를 새로 만든다: `fkFunc`의 대체된 Opcode들을 실행하고, jmp 코드 다음으로 jmp하는 함수이다.
5.  `tgFunc`이 상주하는 PML1의 PFN을 `fkFunc`가 있는 곳으로 조작하고, 권한을 Execute만 준다. (`P1`)
6.  PFN이 `tgFunc`가 있는 페이지이고, 권한이 Read/Write만 존재하는 PML1 Entry를 하나 더 만들어 어딘가에 저장한다. (`P2`)
7.  EPT Violation이 일어날 때 이유에 따라 `P1`과 `P2`를 적절히 교체한다.
8. 후킹 성공! 원래 함수를 호출하고 싶은 경우 `trFunc` 함수를 실행하면 됨

## Conclusion

EPT Hooking은 가히 만능이라 불리는 강력한 기술이니 여러분들도 한 번씩 사용하기 바란다.

글로만 적혀있어 이해가 어려운 분들을 위해 학교 동아리 STEALTH에서 이와 관련해 발표한 자료가 있는데, 이걸 첨부하겠다. [Ref](https://docs.google.com/presentation/d/1A3BoUu-YP-h0uIF29ug04tYnxYSBbMFk6j6Ll6afM9c/edit?usp=sharing)