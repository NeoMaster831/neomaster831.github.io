---
title: "Packer Breaker Pt.1: UPX"
published: 2023-08-01
description: Let's break down UPX, the traditional packer of the world.
tags: [UPX, Packer]
category: Reversing
draft: false
---

:::note[목차]
+ I. 들어가면서..
+ II. Upx?
+ III. 준비
+ IV. 분석
+ V. 유의미한 데이터로의 변환
+ VI. 실전
:::

## I. 들어가면서..

최근 들어 암호학에 관심이 많아졌다. 올해 내가 참여한 CTF의 리버싱 문제에서도 순수 리버싱 실력을 요구하는 문제보다 사고력을 요구하는 리버싱 (예를 들어 역연산, 알고리즘) 문제가 종종 나왔다.

나는 이런 류의 문제를 잘 풀지 못한다. 알고리즘 문제라면 문제 없겠지만, 특히 **암호학적인** 문제가 나온다면 문제가 많다. 그래서 그런 류의 문제를 연습하는 중이다.

하지만 막상 그런 류의 문제만 풀게 되면 실제 리버싱 실력은 늘지 않는다. 그런 문제들은 *"실제 리버싱 세계에서 등장하는 고민인가?"* 와 같은 질문이나 *"이것이 진정 리버싱계에서 필요한 실력인가?"* 와 같은 질문에 *"예."* 라고 대답하기는 어렵다.

리버싱은 '분석하는 학문'이고 분석하는 실력이 리버싱 실력을 척을 가른다고, 암호학적 사고력이 리버싱 실력을 가르지 않는다는 것이 나의 생각이다.

따라서 '진짜 리버싱' 실력을 기르기 위해 나 자신을 성찰하는 면에서 패커를 분석하기로 결심하였다.

## II. Upx?

:::tip
**UPX** is a free, secure, portable, extendable, high-performance **executable packer** for several executable formats.
\- [UPX 공식 사이트](https://upx.github.io/) 중에서
:::

중요한 것은 이 패커가 보안 목적으로 만들어진 패커가 아니라 실제로 실용성을 위해 제작되었다는 것이다. 설명에 'secure'라는 단어가 들어가긴 하다만, 그냥 마케팅을 위해 작성된 문구같다. 그도 그럴 것이, 바이너리 내에서 언패킹 옵션이 존재하여 언패킹이 가능하다!

타 보안 목적 패커 ([Themida](https://www.oreans.com/themida.php), [VMProtect](https://vmpsoft.com/), [Obsidium](https://www.obsidium.de/home)) 과는 다르게 분석이 쉽다는 장점이 있어 패커 분석에 활용되는 패커 중 제일 대표적이다.

이렇게 패커를 분석하고 타파하는 방법을 익히게 되면 나중에 Runtine-VM을 쉽게 분석할 수 있다.

## III. 준비

우선 upx를 다운로드해야 한다. [여기](https://github.com/upx/upx/tags)에서 다운로드 할 수 있다. 본 문서에서는 버전 4.2.4를 사용하였다.

그리고 사용할 프로그램은 두 가지이다. 먼저 첫 번째로 간단하게 "Hello World!"를 출력하는 프로그램을 사용할 것이다.

**sample1.c**
```c
#include <stdio.h>

int main() {
    char* string = "Hello World!";
    printf("%s\n", string);
    return 0;
}
```

[여기](https://drive.google.com/file/d/1LyRN78HTApe_baI3ptsY7R_GuIWAmqUe/view)서 컴파일된 바이너리를 다운로드받을 수 있다.

Upx는 패킹하는 것 외의 작업을 수행하지 않으므로, 패킹을 풀고 처음으로 원본 코드를 실행을 하는 부분 (Original Entry Point, **OEP**) 을 찾아서 그곳에서 덤프를 뜨게 되면 원본 데이터가 나오게 된다. 따라서 우리의 목표는 일단 OEP를 찾는 것이다.

우선 이 원본의 Entry Point 부근 코드를 보자. 나중에 이것이 원본 코드인지 아닌지 확인할 때 도움이 될 것이다.

<img src="/upx/ZeuxMwa.png">

이제 패킹해보자. 패킹하면 다음과 같은 결과가 나온다.

```
PS C:\Users\last_\OneDrive\바탕 화면\Hacking\PackerResearch\upx> .\upx.exe -o sample1_packed.exe .\sample1.exe
                       Ultimate Packer for eXecutables
                          Copyright (C) 1996 - 2024
UPX 4.2.4       Markus Oberhumer, Laszlo Molnar & John Reiser    May 9th 2024

        File size         Ratio      Format      Name
   --------------------   ------   -----------   -----------
     40764 ->     26428   64.83%    win32/pe     sample1_packed.exe

Packed 1 file.
PS C:\Users\last_\OneDrive\바탕 화면\Hacking\PackerResearch\upx>
```

[여기](https://drive.google.com/file/d/12nodhxr4VnaV8D94BWz0ozfeTrakU7gU/view)서 sample1_packed.exe를 다운로드할 수 있다.

이제 준비는 끝났다! 한 번 분석해보자.

## IV. 분석

PE 파일 구조를 분석하게 되면 우선 `.text`, `.data`와 같은 섹션이 사라지고 `.UPX0`, `.UPX1`, `.UPX2` 세 개의 섹션이 생겼다.

주목할 점은, Directory Entry의 Import에 UPX2 섹션이, UPX1 섹션이 TLS로 등록되어 있는 것을 볼 수 있다.

<img src="https://i.imgur.com/7q3XlGs.png">

새로운 EP는 `0x412100`인데, TLS Callback 함수가 EP보다 먼저 실행되기 때문에 TLS Callback 함수의 주소: `0x4122a0`을 먼저 볼 필요가 있다.

<img src="https://i.imgur.com/1pCX9Zb.png">

보니 딱히 아무것도 안 하고 넘어가서 그냥 EP를 분석하기로 했다.

EP를 보니 무언가를 복호화하는 루틴이 보인다.

```c
// ...
  v73 = &v77;
  v71 = a3;
  v70 = a2;
  v69 = a1;
  v4 = byte_40F015;
  v5 = &byte_40F015[-57365];
  v68 = &byte_40F015[-57365];
  while ( 1 )
  {
    v9 = *(_DWORD *)v4;
    v7 = (unsigned int)v4 < 0xFFFFFFFC;
    v4 += 4;
    v10 = v7 + v9;
    v7 = __CFADD__(v7, v9) | __CFADD__(v9, v10);
    v8 = v9 + v10;
    do
    {
      if ( v7 )
      {
        v6 = *v4++;
        *v5++ = v6;
      }
      else
      {
        v11 = 1;
        while ( 1 )
        {
          v12 = __CFADD__(v8, v8);
          v13 = 2 * v8;
          if ( !v13 )
          {
            v14 = *(_DWORD *)v4;
            v7 = (unsigned int)v4 < 0xFFFFFFFC;
            v4 += 4;
            v15 = v7 + v14;
            v12 = __CFADD__(v7, v14) | __CFADD__(v14, v15);
            v13 = v14 + v15;
          }
          v11 += v12 + v11;
          v7 = __CFADD__(v13, v13);
          v16 = 2 * v13 == 0;
          v8 = 2 * v13;
          if ( v7 )
          {
            if ( !v16 )
              break;
            v17 = *(_DWORD *)v4;
            v7 = (unsigned int)v4 < 0xFFFFFFFC;
            v4 += 4;
            v18 = v7 + v17;
            v7 = __CFADD__(v7, v17) | __CFADD__(v17, v18);
            v8 = v17 + v18;
            if ( v7 )
              break;
          }
        }
        v7 = v11 < 3;
        v19 = v11 - 3;
        if ( !v7 )
        {
          v20 = v19 << 8;
          LOBYTE(v20) = *v4++;
          v21 = ~v20;
          if ( !v21 )
          {
            v44 = v68;
            v45 = v68;
            v46 = 275;
            while ( 1 )
            {
              v47 = *v45++;
              v48 = v47 + 24;
              while ( v48 <= 1u && *v45 == 2 )
              {
                v49 = *(_DWORD *)v45;
                LOWORD(v49) = BYTE1(*(_DWORD *)v45);
                v50 = __ROL4__(v49, 16);
                v51 = v50;
                LOBYTE(v50) = BYTE1(v50);
                BYTE1(v50) = v51;
                v52 = v45[4] + 24;
                *(_DWORD *)v45 = &v44[v50 - (_DWORD)v45];
                v45 += 5;
                v48 = v52;
                if ( !--v46 )
                {
                  v53 = v44 + 0x10000;
LABEL_38:
                  v54 = *(_DWORD *)v53;
                  if ( *(_DWORD *)v53 )
                  {
                    v55 = (int *)&v44[*((_DWORD *)v53 + 1)];
                    v53 += 8;
                    v56 = (*((int (__cdecl **)(char *, int, int, int, int, int *, int, int, int))v44 + 18447))(
                            &v44[v54 + 73728],
                            v69,
                            v70,
                            v71,
                            v72,
                            v73,
                            v74,
                            v75,
                            v76);
                    while ( 1 )
                    {
                      v57 = *v53++;
                      if ( !v57 )
                        goto LABEL_38;
                      v58 = v53;
                      v67 = v53;
                      v59 = v57 - 1;
                      do
                      {
                        if ( !v58 )
                          break;
                        v16 = *v53++ == (char)v59;
                        --v58;
                      }
                      while ( !v16 );
                      v60 = (*((int (__cdecl **)(int, char *))v44 + 18449))(v56, v67);
                      if ( !v60 )
                        break;
                      *v55++ = v60;
                    }
                    v54 = (*((int (**)(void))v44 + 18448))();
                  }
                  v61 = (void (__cdecl *)(char *, int, int, int *))*((_DWORD *)v44 + 18450);
                  v62 = v44 - 4096;
                  v66 = v54;
                  v61(v44 - 4096, 4096, 4, &v66);
                  v62[415] &= ~0x80u;
                  v62[455] &= ~0x80u;
                  v61(v44 - 4096, 4096, v63, &v63);
                  v44[70305] = 0;
                  ((void (__cdecl *)(char *, int, _DWORD))(v44 + 70304))(v44 - 4096, 1, 0);
                  do
                    v64 = 0;
                  while ( &v64 != &v65 - 32 );
                  JUMPOUT(0x4012E0); // What?
                }
              }
            }
          }
          a4 = v21;
        }
        v22 = __CFADD__(v8, v8);
        v23 = 2 * v8;
        if ( !v23 )
        {
          v24 = *(_DWORD *)v4;
          v7 = (unsigned int)v4 < 0xFFFFFFFC;
          v4 += 4;
          v25 = v7 + v24;
          v22 = __CFADD__(v7, v24) | __CFADD__(v24, v25);
          v23 = v24 + v25;
        }
        v26 = v22;
        v27 = __CFADD__(v23, v23);
        v8 = 2 * v23;
        if ( !v8 )
        {
          v28 = *(_DWORD *)v4;
          v7 = (unsigned int)v4 < 0xFFFFFFFC;
          v4 += 4;
          v29 = v7 + v28;
          v27 = __CFADD__(v7, v28) | __CFADD__(v28, v29);
          v8 = v28 + v29;
        }
        v30 = v26 + v27 + v26;
        if ( !v30 )
        {
          v31 = 1;
          while ( 1 )
          {
            v32 = __CFADD__(v8, v8);
            v33 = 2 * v8;
            if ( !v33 )
            {
              v34 = *(_DWORD *)v4;
              v7 = (unsigned int)v4 < 0xFFFFFFFC;
              v4 += 4;
              v35 = v7 + v34;
              v32 = __CFADD__(v7, v34) | __CFADD__(v34, v35);
              v33 = v34 + v35;
            }
            v31 += v32 + v31;
            v7 = __CFADD__(v33, v33);
            v36 = 2 * v33 == 0;
            v8 = 2 * v33;
            if ( v7 )
            {
              if ( !v36 )
                break;
              v37 = *(_DWORD *)v4;
              v7 = (unsigned int)v4 < 0xFFFFFFFC;
              v4 += 4;
              v38 = v7 + v37;
              v7 = __CFADD__(v7, v37) | __CFADD__(v37, v38);
              v8 = v37 + v38;
              if ( v7 )
                break;
            }
          }
          v30 = v31 + 2;
        }
        v39 = (a4 < 0xFFFFF300) + v30 + 1;
        v40 = &v5[a4];
        if ( a4 <= 0xFFFFFFFC )
        {
          do
          {
            v42 = *(_DWORD *)v40;
            v40 += 4;
            *(_DWORD *)v5 = v42;
            v5 += 4;
            v43 = v39 <= 4;
            v39 -= 4;
          }
          while ( !v43 );
          v5 += v39;
        }
        else
        {
          do
          {
            v41 = *v40++;
            *v5++ = v41;
            --v39;
          }
          while ( v39 );
        }
      }
      v7 = __CFADD__(v8, v8);
      v8 *= 2;
    }
    while ( v8 );
  }
}
```

`v4`에 `byte_40F015`의 주소, `v5`와 `v68`에 각각 `byte_40F015 - 0xE015`의 주소를 넣는다. 그리고 이것은 섹션 `.UPX0`의 시작과 같다. 아마 아무것도 없는 `.UPX0`의 터무니없는 크기는 복호화된 데이터를 저장하는 용도로 추측할 수 있다.

이 루틴을 직접 분석하기에는 다소 무리가 있으므로 `UPX0` (`0x401000`) 주변의 메모리 변화로 알아보면, 제대로 되진 않았지만 바이트코드처럼 보이는 부분을 볼 수 있다.

```
UPX0:00401000 sub     esp, 1Ch
UPX0:00401003 mov     eax, [esp+20h]
UPX0:00401007 mov     eax, [eax]
UPX0:00401009 mov     eax, [eax]
UPX0:0040100B cmp     eax, 0C0000091h
UPX0:00401010 ja      short loc_401060
UPX0:00401012 cmp     eax, 0C000008Dh
UPX0:00401017 jnb     short loc_401079
UPX0:00401019 cmp     eax, 0C0000005h
UPX0:0040101E jnz     loc_4010F0
UPX0:00401024 mov     dword ptr [esp+4], 0
UPX0:0040102C mov     dword ptr [esp], 0Bh
UPX0:00401033 call    near ptr 446A103Ah
UPX0:00401038 cmp     eax, 1
UPX0:0040103B jz      loc_401189
UPX0:00401041 test    eax, eax
UPX0:00401043 jnz     loc_401130
UPX0:00401049 lea     esi, [esi+0]
```

이렇게 복호화된 곳으로 Jump하는 명령어가 있을 것이라 추측하고 `jmp` Opcode를 중점적으로 찾아보면, 무려 우리가 패킹을 안 했을 때의 EP인 `0x4012e0`으로 점프하는 코드를 찾을 수 있다.

```
UPX1:0041228A push    ebx
UPX1:0041228B call    ecx
UPX1:0041228D popa
UPX1:0041228E lea     eax, [esp+38h+var_B8]
UPX1:00412292
UPX1:00412292 loc_412292:                             ; CODE XREF: start+196↓j
UPX1:00412292 push    0
UPX1:00412294 cmp     esp, eax
UPX1:00412296 jnz     short loc_412292
UPX1:00412298 sub     esp, 0FFFFFF80h
UPX1:0041229B jmp     near ptr dword_4012E0
```

이는 디컴파일된 코드에서 `JUMPOUT(0x4012E0)` 의 구문과 같다. 여기의 주소 `0x41229B`에 BP를 설치하고 넘어가보면 다음과 같이 성공적으로 OEP를 찾을 수 있다.

<img src="https://i.imgur.com/IajKlB0.png">

실제로 원래 바이트코드였던 `83 EC 1C C7 04 24 ...` 를 발견할 수 있다. 이로써 우리는 OEP를 발견해내었다. 여기서부터 프로그램을 디버깅하는 등의 일반적인 원본 프로그램에서 할 수 있던 행위를 모두 할 수 있어졌다.

## V. 유의미한 데이터로의 변환

아까 `UPX0` 섹션에 복호화된 데이터가 저장되는 것을 보았으니 우리는 프로세스를 덤핑하여서 원하는 부분을 뽑을 수 있다. 방법은 다양한데, 나는 [Scylla](https://www.unknowncheats.me/forum/general-programming-and-reversing/184679-scylla-0-9-8-x86-x64.html) 로 덤핑하였다.

이렇게 해서 생성된 덤프 파일은 [여기](https://drive.google.com/file/d/1gYLLSzsQWO9GMG0LiGPMYg5PJ6NODwDQ/view) 있다.

실제로 `0x401460` 부분을 보면 다음과 같은 함수가 있다. 이것은 우리가 작성한 프로그램으로, 분석한 결과 얻고 싶었던 것을 얻었으니 이 분석은 성공했다고 볼 수 있다.

*In Ida,*
```c
int sub_401460()
{
  sub_4019C0();
  puts(aHelloWorld);
  return 0;
}
```

## VI. 실전

인생은 실전이다. 한 번 다른 프로그램에서 시도해보자. 두 번째 프로그램은 [여기](https://drive.google.com/file/d/1Rx2FaMFLvXUvVlETM57F2ggX0Irvw2Ri/view) 있다.