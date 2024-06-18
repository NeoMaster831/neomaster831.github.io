---
title: Hypervisor Basics
published: 2024-05-16
description: Learn Hypervisor Basics.
tags: [VT-x, Hypervisor, Basics]
category: Hypervisor
draft: false
---

# Hypervisor Basics

HV는 정말 강력하고 좋은 기술이지만, 개념이 정말 어렵고 사람들에게 잘 알려지지 않아 현재 정돈된 문서가 별로 없는 상태이고, 한국어로 정리된 자료는 특히 더 없다. 모든 사람이 HV에 입문하기 쉽게 글을 작성한다.

:::note[이 글, 그리고 다음으로 이어질 글에서 독자는 다음을 알고 있다고 가정한다.]
+ 커널에 대한 기본적인 지식
+ C/C++에 대한 Expert한 지식
+ 컴퓨터의 내부적 동작에 대한 준수한 이해
:::

## 목표

1. HV의 기본 개념과 목표, 가치를 이해한다.
2. CPU가 VMX를 지원하는 지 확인할 수 있다.

## 소개

**Virtual Machine Monitor** (VMM, also known as **HyperVisor**, HV)는 여러 운영 체제를 가상화시켜 하나의 시스템에서 작동시키고 그것을 감독하는 기술을 말한다.

이 기술을 이용하는 상용 프로그램으로는 [VMware](https://www.vmware.com/), [VirtualBox](https://www.virtualbox.org/), 더 광범위하게 바라본다면 Microsoft의 [Hyper-V](https://learn.microsoft.com/ko-kr/virtualization/hyper-v-on-windows/about/), Linux의 [Xen](https://xenproject.org/)이 있다.

이 가상화 기술을 어디에 쓰냐 하면, 한 운영체제의 커널보다 위의 Previlege에서 원초적인 작업 (예를 들어 메모리 접근)을 감독하고 제어하고 싶을 경우 사용하는 기술이다. 가령 HV는 PatchGuard가 보호하는 Syscall Table을 Hooking할 수 있는데, 이는 HV가 PG나 커널보다 높은 Previlege에서 동작하기 때문이다.

<img src="https://miro.medium.com/v2/resize:fit:400/format:webp/1*GB-uW-xbSIHHkkH4eSVKaQ.jpeg">

중요한 점은 CPU 또한 이 HV 기술을 지원하는데, Intel의 VT-x, AMD의 AMD-V가 있다. 우리는 이 기술을 활용하여 사용자의 컴퓨터를 가상화시킬 수 있고, 사용자가 커널 위에서 나의 컴퓨터에 대한 동작을 제어하게 할 수 있는 프로그램을 제작하는 것이 목표이다.

여기서 다룰 것은 아쉽지만 AMD 계열 CPU가 아닌, Intel 계열 CPU의 VT-x 기술을 활용하여 HV를 제작하는 것이다.

## 단어들

+ Host (PC) / Guest (PC)
	+ Host: Guest 위에서 Guest를 감독하고 제어하는 객체이다.
	+ Guest: Host 밑에서 제한된 동작을 수행하는 객체이다.
+ VMX
	+ VT-x 기술에 사용되는 *Internal*적인 기술이다. 보통 VMX Operation이라고 불리는데, 이는 VMX가 지원되는 CPU에서 특별히 추가된 Opcode 조합들이다.
		+ 예를 들어, vmxon, vmxoff, vmptrld, vmcall, ... 등이 있다.


## 소개 2

우선 HV의 목표는 Guest가 실제 가상화된 환경에서 동작하는 것을 **알아차리지 못하게 하고** 접근을 제한하는 것이다.

그래서, Guest가 가상화된 환경에서 동작하는지 알 수 있는 방법은 **굉장히 제한적이다**. 사실 공식적인 방법으로 알 수 없지만, [rdtsc timing check](https://secret.club/2020/01/12/battleye-hypervisor-detection.html)와 같은 꼼수로 알 수 있다. 아무튼, 위와 같은 이유로 HV는 커널에서 작동하는 대부분의 안티치트를 우회할 수 있다. 그래서 게임 핵을 만들 때 간간히 이용되는 기술이지만 말했듯 기술 자체가 굉장히 어려워서 사용자들이 이용하지 않는다.

## End

앞으로 Hypervisor에 관한 문서들이 업데이트 될 것이다. 그것의 종합적인 소스 코드는 [이 레포지토리](https://github.com/fukurei/Reisen6900X/)에서 확인이 가능하다. 지속적인 업데이트로 소스 코드가 변경될 수 있다.있다.
