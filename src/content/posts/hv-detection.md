---
title: Hypervisor basics, pt. 3
published: 2024-11-10
description: 'Detecting hypervisors in the wild'
image: '/hv-detection/detect.png'
tags: [Hypervisor, Detection]
category: 'Hypervisor'
draft: false 
---

# Detection

Hypervisor Detection에 대해 알아보자. Microsoft의 Hyper-V나 VMware, VirtualBox와 같은 대형 하이퍼바이저를 잡는 목적이 아니다. 그런 하이퍼바이저들은 합법적일 뿐더러 소개할 방법에 감지되지 않는다.

우리가 잡는 하이퍼바이저는 불순한 의도(게임 핵, 바이러스, 등등..)를 가지는 하이퍼바이저에 대한 감지를 수행하는 것이다.

애초에 HV가 Guest한테 감지되지 않을 목적으로 제작한 것이라 할 수 있는 방법이 거의 없다시피 한다. 전에 소개한 EPT Hooking같은 경우는 잡을 수 있는 방법이 아예 없다. 따라서 우리는 편법으로 잡을 수밖에 없다. 여기선 대표적인 Detection 3개를 소개한다.

## [0x1] RDTSC Timing Check

- 가장 대표적인 Hypervisor Detection이다. VM은 기본적으로 속도가 굉장히 느린데, 이를 근거로 `rdtsc` 명령을 이용해 CPU 사이클을 체크한다. 여기서 특정 Threshold를 초과하면 VM을 사용하는 것으로, 아니면 베어메탈로 판정한다.
- 장점: 체크하기 쉽다. 몇 줄의 코드로 가능하다.
- 단점: 뚫기가 쉽고, 부정확하다.

```c
void performRDTSCTimingCheck(){
    unsigned long long int time1, time2, sum = 0;
    const unsigned char avg = 100;
    
    for (int i = 0; i < avg; i++){
        time1 = __rdtsc();
#ifdef _WIN32
        __asm cpuid;
#elif __linux__
        __asm__ volatile("CPUID"::: "eax", "ebx", "ecx", "edx", "memory");
#endif
        time2 = __rdtsc();
        sum += time2 - time1;
    }

    sum = sum / avg;
    
    printf("Ticks on average: %llu\n", sum);

    if(sum > 500){
        puts("Probably a VM");
    }else{
        puts("Probably Bare-Metal");
    }
}
```

- Bypass: `rdtsc` 명령어는 VM Exit을 유발하는 명령어이다. 그 말은 즉슨, VM Exit Handler에서 핸들링할 수 있다는 것이다. 따라서 `rdtsc` 명령어의 반환값을 조작하여 판정을 바꿀 수 있다.

## [0x2] MSR Timing Check

- RDTSC Timing Check랑 거의 같다. `APERF` MSR Entry를 통해 CPU 사이클을 측정할 수 있는데 이를 통해 감지하는 방법이다.
- 장점: RDTSC VM Exit에 걸리지 않는다. 체크하기 쉽다.

```c
BOOLEAN
APERFMsrTimingCheck()
{
    KAFFINITY new_affinity = {0};
    KAFFINITY old_affinity = {0};
    UINT64 old_irql = 0;
    UINT64 aperf_delta = 0;
    UINT64 aperf_before = 0;
    UINT64 aperf_after = 0;
    INT cpuid_result[4];

    /*
     * First thing we do is we lock the current thread to the logical
     * processor its executing on.
     */
    new_affinity = (KAFFINITY)(1ull << KeGetCurrentProcessorNumber());
    old_affinity = ImpKeSetSystemAffinityThreadEx(new_affinity);

    /*
     * Once we've locked our thread to the current core, we save the old
     * irql and raise to HIGH_LEVEL to ensure the chance our thread is
     * preempted by a thread with a higher IRQL is extremely low.
     */
    old_irql = __readcr8();
    __writecr8(HIGH_LEVEL);

    /*
     * Then we also disable interrupts, once again making sure our thread
     * is not preempted.
     */
    _disable();

    /*
     * Once our thread is ready for the test, we read the APERF from the
     * MSR register and store it. We then execute a CPUID instruction
     * which we don't really care about and immediately after read the APERF
     * counter once again and store it in a seperate variable.
     */
    aperf_before = __readmsr(IA32_APERF_MSR) << 32;
    __cpuid(cpuid_result, 1);
    aperf_after = __readmsr(IA32_APERF_MSR) << 32;

    /*
     * Once we have performed our test, we want to make sure we are not
     * hogging the cpu time from other threads, so we reverse the initial
     * preparation process. i.e we first enable interrupts, lower our irql
     * to the threads previous irql before it was raised and then restore
     * the threads affinity back to its original affinity.
     */
    _enable();
    __writecr8(old_irql);
    ImpKeRevertToUserAffinityThreadEx(old_affinity);

    /*
     * Now the only thing left to do is calculate the change. Now, on some
     * VMs such as VMWARE the aperf value will be 0, meaning the change will
     * be 0. This is a dead giveaway we are executing in a VM.
     */
    aperf_delta = aperf_after - aperf_before;

    return aperf_delta == 0 ? TRUE : FALSE;
}
```

- Bypass: 얘도 사실 MSR Read를 이유로 VM Exit을 한다. APERF MSR를 읽는 경우를 한정해서 반환값을 다르게 하면 뚫린다.

## [0x3] INVD Emulation

- INVD 또한 VM Exit을 유발하는 명령어다. 이걸 Handling하는게 여간 빡센 게 아니고, 이게 진짜 쓸모없어서 대형 HV가 아니면 구현을 안 한다. 이걸 노린 Detection 방법이다.
- 장점: 이걸 뚫기 어렵다.
- 단점: 알면 뚫린다.

```nasm
.code

; Tests the emulation of the INVD instruction
;
; source and references:
;
; https://secret.club/2020/04/13/how-anti-cheats-detect-system-emulation.html#invdwbinvd
; https://www.felixcloutier.com/x86/invd
; https://www.felixcloutier.com/x86/wbinvd
;
; Returns int

TestINVDEmulation PROC

	pushfq
	cli
	push 1					; push some dummy data onto the stack which will exist in writeback cache
	wbinvd					; flush the internal cpu caches and write back all modified cache 
							; lines to main memory
	mov byte ptr [rsp], 0	; set our dummy value to 0, this takes place inside writeback memory
	invd					; flush the internal caches, however this instruction will not write 
							; back to system memory as opposed to wbinvd, meaning our previous 
							; instruction which only operated on cached writeback data and not
							; system memory has been invalidated. 
	pop rax					; on a real system as a result of our data update instruction being
							; invalidated, the result will be 1. On a system that does not
							; properly implement INVD, the result will be 0 as the instruction does
							; not properly flush the caches.
	xor rax, 1				; invert result so function returns same way as all verification methods
	popfq
	ret

TestINVDEmulation ENDP

END
```

- Bypass: INVD VM Exit을 핸들링한다.