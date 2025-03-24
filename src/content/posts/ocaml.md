---
title: General Approach for OCaml binaries
published: 2025-03-24
description: 'Break down functional programming language'
image: ''
tags: [ 'OCaml' ]
category: 'Reversing'
draft: false 
---

> OCaml 빌드를 위해 [dune](https://dune.build/)을 이용하였습니다.
> 분석을 위해 IDA Professional 9.0 Beta를 사용하였습니다.
> 분석을 위해 Binary Ninja 4.2 유료 버전을 사용하였습니다.
> OS는 Ubuntu 22.04 (Linux)를 사용하였습니다.

## OCaml?

OCaml은 비주류 언어로, 함수 언어 패러다임을 준수하는 언어입니다. 함수형 언어 치고는 확장성과 기능이 F#보다 딸리고, 그렇다고 컴파일 해서 쓰기에는 너무 느리고, ... 와 같은 다양한 이유로 OCaml은 시장에서 위치를 잡지 못하고 비주류 언어로 전락하게 되었습니다.

### To do with reversing

함수형 언어라는 패러다임에 걸맞게 [CPS](https://en.wikipedia.org/wiki/Continuation-passing_style)적인 면모를 보여주는 것도 모자라서, 비주류 언어라 자료의 양이 부족합니다. 이로 인해 OCaml 바이너리의 리버싱은 최고 난이도로 자리잡았습니다.

### How?

이러한 과제가 주어진 경우 다양한 소스 파일을 컴파일하며 컴파일러 상에서 공통적으로 발견되는 부분을 종합하며 귀납적으로 추론하는 방법이 가장 효과적입니다. 마치, C++, Rust, Go와 같은 언어에서 주로 등장하는 접근법이죠.

# [1] Basic Program

```ocaml
let () = print_endline  "Hello, World!"
```

다음 프로그램을 분석해 봅시다. 우선 분석하게 되면 `strip`을 하지 않는 이상, **사용자가 의도한 Entry Point는 `camlDune__exe__Main.entry`** 에서 시작합니다.

```c
__int64 __fastcall camlDune__exe__Main_code_begin(int a1, int a2, int a3, int a4, int a5, int a6)
{
  __int64 v6; // r14
  char v8; // [rsp+0h] [rbp-140h] BYREF

  if ( (unsigned __int64)&v8 < *(_QWORD *)(v6 + 40) )
    caml_call_realloc_stack(a1, a2, a3, a4, a5, a6, required_space: 0x21uLL);
  camlStdlib_print_endline_369();
  return 1LL;
}
```

디컴파일한 결과인데, 보면 `Hello World`와 같은 문자열이 보이지 않는 것을 볼 수 있습니다. 이는 어셈블리 결과를 보면 의문이 해결됩니다.

```nasm
loc_17DAE:
lea     rax, camlDune__exe__Main_1 ; "Hello, World!"
call    camlStdlib_print_endline_369
```

Calling Convention이 일반적으로 알고 있는 Linux의 `rdi`, `rsi`, ... 혹은 Windows의 `rdx`, `rcx`, ... 의 `__fastcall` 규칙과 다른 것을 볼 수 있습니다.

알아낸 사소한 사실은, IDA보다 Binary Ninja가 분석하기 더 용이하다는 점입니다.

# [2] Basic Functional Program

이번엔 몇 개의 독립적인 함수를 호출해 보겠습니다.

```ocaml
let foo(x, y) =
  x + y
let bar(x, y) = 
  x * y

let () =
  print_endline(string_of_int(foo(1, 2)));
  print_endline(string_of_int(bar(6, 9)))
```

함수 목록에는 `.foo_270`, `.bar_275`, `.entry` 가 있으며, `entry`를 제외한 함수에 임의의 숫자를 붙여 맹글링하는 것 같습니다.

`.foo`
```il
00017dc0    int64_t camlDune__exe__Main.foo_270(int64_t arg1 @ rax, int64_t arg2 @ rbx) __pure
00017dc5        return arg1 + arg2 - 1
```

`.bar`
```il
00017dd0    int64_t camlDune__exe__Main.bar_275(int64_t arg1 @ rax, int64_t arg2 @ rbx) __pure
00017ddd        return (arg1 - 1) * (arg2 s>> 1) + 1
```

위와 같이 두 함수는 그다지 중요한 부분이 없으나, 엔트리 부분에서 주목해야 할 사항이 있습니다.

```il
...
00017df2  488d35877d0400     lea     rsi, [rel camlDune__exe__Main.4]
00017df9  488d3dc07d0400     lea     rdi, [rel camlDune__exe__Main]
00017e00  4889e3             mov     rbx, rsp {__return_addr}
00017e03  498b6640           mov     rsp, qword [r14+0x40]
00017e07  e814820100         call    caml_initialize
00017e0c  4889dc             mov     rsp, rbx
00017e0f  488d358a7d0400     lea     rsi, [rel camlDune__exe__Main.3]
00017e16  488d3da37d0400     lea     rdi, [rel camlDune__exe__Main]
00017e1d  4883c708           add     rdi, 0x8  {data_5fbc8}
00017e21  4889e3             mov     rbx, rsp {__return_addr}
00017e24  498b6640           mov     rsp, qword [r14+0x40]
00017e28  e8f3810100         call    caml_initialize
...
```

정의한 함수의 수, 두 개만큼의 데이터를 배열 비슷한 형태로 하드코딩을 해 놓은 것을 볼 수 있었습니다: `camlDune__exe__Main`, `data_5fbc8`. 또한 이와 매칭되는 `camlDune__exe__Main.4`, `camlDune__exe__Main.3` 과 같은 심볼도 확인할 수 있었습니다.

```il
0005fb80  void* camlDune__exe__Main.4 = caml_tuplify2
0005fb88                          07 00 00 00 00 00 00 fe          ........
0005fb90  void* data_5fb90 = camlDune__exe__Main.foo_270
0005fb98                          f7 0f 00 00 00 00 00 00          ........
0005fba0  void* camlDune__exe__Main.3 = caml_tuplify2
0005fba8                          07 00 00 00 00 00 00 fe          ........
0005fbb0  void* data_5fbb0 = camlDune__exe__Main.bar_275
0005fbb8                          00 0b 00 00 00 00 00 00          ........
0005fbc0  int64_t camlDune__exe__Main = 0x1
0005fbc8  int64_t data_5fbc8 = 0x1
```

다음으로는 함수들을 호출하는데, 여기서 `rdi`는 `caml_tuplify2`로, `caml_tuplify2`는 함수 Wrapper의 역할을 하며 사용자 정의 함수로 리다이렉트 합니다.

```il
.text:000055555556BE2D mov     rsp, rbx
.text:000055555556BE30 lea     rax, camlDune__exe__Main
.text:000055555556BE37 mov     rbx, [rax]
.text:000055555556BE3A lea     rax, camlDune__exe__Main_1
.text:000055555556BE41 mov     rdi, [rbx]
.text:000055555556BE44 call    rdi
.text:000055555556BE46 call    camlStdlib_string_of_int_175
.text:000055555556BE4B call    camlStdlib_print_endline_369
.text:000055555556BE50 lea     rax, camlDune__exe__Main
.text:000055555556BE57 mov     rbx, [rax+8]
.text:000055555556BE5B lea     rax, camlDune__exe__Main_2
.text:000055555556BE62 mov     rdi, [rbx]
.text:000055555556BE65 call    rdi
.text:000055555556BE67 call    camlStdlib_string_of_int_175
.text:000055555556BE6C call    camlStdlib_print_endline_369
.text:000055555556BE71 mov     eax, 1
.text:000055555556BE76 retn
```

함수의 Calling Convention 또한 `rax`, `rbx`로 되는 것을 확인하였습니다.

신기한 점은 `foo` 함수의 아웃풋이 7이라는 점입니다. 예측한 건 3인데, 7이라는 점이 혼란을 가중시켰습니다.

그리고, `Main.1` 및 `Main.2` 에 함수의 인자가 들어가 있었습니다.

```il
0005fbe8  camlDune__exe__Main.2:
0005fbe8                          0d 00 00 00 00 00 00 00          ........
0005fbf0  13 00 00 00 00 00 00 00 00 0b 00 00 00 00 00 00  ................
0005fc00  camlDune__exe__Main.1:
0005fc00  03 00 00 00 00 00 00 00 05 00 00 00 00 00 00 00  ................
0005fc10  00 00 00 00 00 00 00 00                          ........
```

# [3] Getting User Input

최적화 이슈인 것인가 궁금하여 일단 User Input을 받는 프로그램을 작성해 보았습니다.

```ocaml
let foo(x, y) = 
  x + y
let () =
  let a = read_int() in
  let b = read_int() in
  print_endline(string_of_int(foo(a, b)))
```

여기서도 똑같이 `1`이 `3`으로, `2`가 `5`로, 등등 **모든 상수 숫자 값이 $2n + 1$로 나타나는 것을 확인했습니다.**

찾아보니 이것은 Tagged Representation으로, 가장 하위 비트가 '원시 요소'일 경우 1, 아닐 경우 (String과 같은 포인터나 복잡한 구조체) 0으로 나타내는 것이였습니다.

모든 자료형 (심지어는 byte도!) 은 64바이트 혹은 32바이트로 나타내어집니다. 이것이 향후 리버싱에서 중요한 점을 담당합니다.

더 작위적인 예시는 다음과 같습니다.

```ocaml
let () =
  let a = read_line() in
  print_char a.[0]
```

string 자료형 안의 데이터 자체는 utf-8으로 저장됩니다. 하지만 `a.[0]`과 같이 데이터를 꺼낼 경우 다음과 같이 초기화되게 됩니다.

```il
000055555556bdd6  480fb600           movzx   rax, byte [rax]
000055555556bdda  488d440001         lea     rax, [rax+rax+0x1]
000055555556bddf  e87c200000         call    camlStdlib.print_char_354
```

이 역시 $2n+1$의 공식을 따르고 있음을 알 수 있습니다.

# [4] Recursive Calling

이런 식의 프로그램을 작성했습니다.

```ocaml
let c(k) =
  let x = read_int() in
  k + x

let b(k) =
  let x = read_int() in
  k + c(x)

let a() =
  let x = read_int() in
  b(x)

let () =
  print_endline (string_of_int (a()))
```

실행하는 부분을 보게 되면, 최적화가 실행된 모습을 볼 수 있습니다.

```il
00017eda  4889dc             mov     rsp, rbx
00017edd  b801000000         mov     eax, 0x1
00017ee2  e8d9230000         call    camlStdlib.read_int_399
00017ee7  e8f4feffff         call    camlDune__exe__Main.b_274
00017eec  e8df120000         call    camlStdlib.string_of_int_175
00017ef1  e85a210000         call    camlStdlib.print_endline_369
00017ef6  b801000000         mov     eax, 0x1
00017efb  c3                 retn     {__return_addr}
```

최적화를 수행했더라고 해도 아마 함수 안에서 다른 함수를 호출하는 것은 유지되고 (최소한 모든 것이 인라인으로 바뀌는 것이 아니라 다행입니다..) , 함수가 인라인 처리되었다고 해도 심볼과 함수 본문이 남아있다는 것을 알 수 있었습니다. (물론, `caml_initialize` 또한 3번 호출합니다.)

# [5] List

다음과 같은 프로그램을 만들었습니다.

```ocaml
let list1 = [1; 2; 3; 4; 5]
let list2 = [6; 7; 8; 9; 10]

let () =
  let list3 = list1 @ list2 in
  print_endline (string_of_int (List.length list3));
```

List는 기본적으로 다음 구조로 되어 있습니다.
```c
struct List_Entry {
  UINT64 Value;
  List_Entry* Next;
  // ...
}
```

또한 굉장히 헤맸던 것인데, `$40` 이라는 함수가 List와 List를 합치는 함수였습니다. 이 외에도 `$5e`, 등 함수가 있었는데, 이는 연산자 ASCII를 변환한 것으로 생각하면 됩니다. 이 외에는 함수 이름이 직관적이라서 처음 봐도 알 수 있었습니다.

## [5-1] String List

다음과 같은 프로그램을 만들었습니다.

```ocaml
let list1 = ["a"; "b"; "c"; "d"; "e"]
let list2 = ["f"; "g"; "h"; "i"; "j"]

let () =
  let list3 = list1 @ list2 in
  print_endline (String.concat " " list3);
```

이유는 String은 Value에 어떻게 들어갈까 궁금해서 넣었습니다. 결과는 '참'입니다.

# Extra

OCaml의 오브젝트들은 거의 모두 첫 번째 8바이트에 데이터를 저장하고, 부차적인 것들은 그 뒤에 나오며, **거의 모두 8바이트를 따릅니다.**

이 외에 알아둬야 할 것은, `fold_left`, `map`과 같은 함수를 써서 매핑할 때는 `rax` 또는 `rbx`에 호출하는 함수를 같이 넣어서 호출합니다.

> Practice
> 1. [ImaginaryCTF 2024 - Oh, a Camel!](https://github.com/ImaginaryCTF/ImaginaryCTF-2024-Challenges-Public/tree/main/Reversing/oh_a_camel)
> 2. [Calm Lambdas](https://dreamhack.io/wargame/challenges/1790)

# End

이 외에도 함수형 언어의 특징이 몇 개 더 있지만, 핵심적인 것만 정리해 보았습니다. 결국 가장 중요한 것은 리버서의 역량이겠지요. 하지만 알고 가면 좋은, 그런 특징들은 필요하다고 생각하여 여러분들을 위해 정리해 보았습니다.
