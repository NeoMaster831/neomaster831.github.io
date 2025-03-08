---
title: Memory Conversion at Boot Time
published: 2025-03-07
description: 'Physicall Address Conversion on Bootkit'
image: ''
tags: [ UEFI, Physical Memory ]
category: 'Computer Science'
draft: false 
---

*모든 내용은 Windows 11 24H2 기준으로 작성되었습니다.*

[Voyager](https://github.com/backengineering/Voyager)의 소스 코드를 보다가, 신기한 점을 발견했습니다.

```cpp
{
// ...
    auto init() -> vmxroot_error_t;
    auto map_guest_phys(guest_phys_t phys_addr, map_type_t map_type = map_type_t::map_src) -> u64;
    auto map_guest_virt(guest_phys_t dirbase, guest_virt_t virt_addr, map_type_t map_type = map_type_t::map_src) -> u64;

    auto map_page(host_phys_t phys_addr, map_type_t map_type = map_type_t::map_src) -> u64;
    auto get_map_virt(u16 offset = 0u, map_type_t map_type = map_type_t::map_src) -> u64;

    auto translate(host_virt_t host_virt) -> u64;
    auto translate_guest_physical(guest_phys_t guest_phys, map_type_t map_type = map_type_t::map_src) -> u64;
    auto translate_guest_virtual(guest_phys_t dirbase, guest_virt_t guest_virt, map_type_t map_type = map_type_t::map_src) -> u64;

    auto read_guest_phys(guest_phys_t dirbase, guest_phys_t guest_phys, guest_virt_t guest_virt, u64 size) -> vmxroot_error_t;
    auto write_guest_phys(guest_phys_t dirbase, guest_phys_t guest_phys, guest_virt_t guest_virt, u64 size) -> vmxroot_error_t;
    auto copy_guest_virt(guest_phys_t dirbase_src, guest_virt_t virt_src, guest_virt_t dirbase_dest, guest_virt_t virt_dest, u64 size) -> vmxroot_error_t;
// ...
}
```

기본적으로 부트킷은 `ntoskrnl.exe`가 로드되지 않아 Windows NT 함수를 사용할 수 없습니다. 이 말은 즉, 물리 메모리 주소와 가상 메모리 사이의 전환을 유도하는 `MmGetPhysicalAddress` 함수 및 `MmMapIoSpace` 함수와 같은 유틸리티를 사용할 수 없습니다.

물론 최신 윈도우 (Windows 10 1703 이상 버전) 에서는 Creators Update로 `winload.efi` 에 함수가 다수 추가되어 쉬워졌으나, 여전히 어려운 것은 마찬가지입니다.

궁금하여 이를 어떻게 구현하였나 보았는데, 굉장히 심오한 개념이 쓰여 이해하기 정말 어려웠습니다. OS 엔지니어들이 얼마나 괴물같은 분들인지 알 수 있는 대목이였습니다.

이렇게 어려운 개념을 공부 겸 가져와봤습니다.

# Self-Referencing PML4E

CPU의 모드가 Protected Mode 이상으로 결정되며 CPU의 CR3가 설정되면 비로소 가상 주소를 사용할 수 있게 되고 가상 주소를 통해서만 메모리에 접근할 수 있습니다. UEFI 부트 모드는 실행 이전에 BIOS에서 Long Mode로 전환이 완료되었기 때문에, `bootmgfw.efi` 실행 시각에는 이미 CR3이 설정이 되어 있습니다.

CR3에는 PML4, PDPT, PD, PT로 이루어진 4단계 변환 테이블을 이용해 가상 주소를 물리 주소로 매핑합니다. 자세한 내용은 [여기](https://en.wikipedia.org/wiki/Page_table)를 참조하세요.

문제는 CR3에 저장되는 최초의 PML4 테이블은 물리 주소로 저장이 되어 있습니다. 즉, 부트킷 레벨에서는 이러한 물리 메모리 주소에 접근을 하기 어렵습니다.

하지만 하나의 간단한 **트릭**으로 이를 단번에 해결할 수 있습니다. 어떤 **고정된 가상 주소**를 통해 **가상 주소를 물리 주소로 변환**할 수 있다면 믿으시겠습니까? 이 **트릭**이 바로 Self-Referncing PML4E 입니다.

간단합니다. 하나의 PML4E (PML4 Entry) 를 PML4의 물리 주소로 설정하는 것입니다. 보통 이것은 510번째 Entry로, `winload.efi` 가 설정합니다.

우선 PML4 테이블에 접근하는 주소는 `[510][510][510][510]` 입니다. PDPT, PD, PT가 재귀적으로 PML4을 가리키고 있기 때문에, 최종적으로 PML4의 테이블을 가리킵니다. 즉, `0xFFFFFF7FBFDFE000`가 PML4의 가상 주소입니다.

마찬가지로 원하는 PDPT 테이블은 `[510][510][510][x]` 입니다. 생각해보면 알 수 있습니다.
PD 테이블은 `[510][510][x][y]`, PT 테이블은 `[510][x][y][z]` 입니다.

따라서 만약 가상주소 `[a][b][c][d]`에 대한 물리 주소를 참조하려면, `[510][a][b][c]` + Offset `sizeof(PTE) * d` (단, $0$ $\le$ `d` $< 512$) 를 참조하면 됩니다.

<img src="/bootphys/2.png">

# Getting VA by Direct Page Switching

이제 물리 주소를 가상 주소로 매핑하는 과정을 살펴봅시다. 물리 주소를 가상 주소로 '변환'하는 것은 **존재하지 않는다는 점**을 리마인드해 줍시다. 대신, 임의의 가상 주소로 '매핑'하는 것은 가능합니다.

기본적인 아이디어는 매핑할 가상 주소를 미리 **할당**한다는 개념입니다. 즉, 어떤 한 고정된 가상 주소를 변화하는 물리 주소에 대응하는 것으로 만든다 - 입니다.

PTE 하나를 CPU 논리 코어 하나에 대응시키는 것으로, 보통 `src` PTE와 `dest` PTE 두 개를 할당합니다. `src` 는 가상의 데이터를 그대로 물리 메모리에 쓰는 방식으로, **물리 메모리 쓰기** 입니다. `dest`는 물리 메모리를 읽어서 가상 페이지로 로드하는 방식으로, **물리 메모리 읽기** 입니다.

방법론은, 미리 고정한 가상 페이지의 PTE의 `pfn`을 수정하고 가상 페이지의 데이터를 각각 씀, 읽음으로써 이룰 수 있습니다.

`pfn`을 수정하는 것은 Self-Referencing PML4E를 활용하면 할 수 있습니다.

<img src="/bootphys/3.png">

# Conclusion

이와 같은 방법은 따로 메모리 할당이나 함수의 활용 없이 물리 메모리 변환, 물리 메모리 매핑, 읽기, 쓰기를 할 수 있는 면에서 의의를 가집니다.

이를 활용하는 곳은 많지 않지만, 대부분 Type-1 Hypervisor에서 사용합니다. 이들은 운영 체제의 말 그대로 **위**에서 실행되어야 하기 때문에, 운영 체제의 도움을 받지 않고 메모리 변환이 필요합니다. Microsoft의 Hyper-V가 대표적으로 활용하고 있습니다.

### References
1. [Voyager](https://github.com/backengineering/Voyager)
2. [Hyper-V](https://learn.microsoft.com/ko-kr/windows-server/virtualization/hyper-v/hyper-v-overview)
3. [Napoca](https://github.com/bitdefender/napoca)
