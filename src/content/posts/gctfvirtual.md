---
title: Google CTF Write-Ups
published: 2024-06-26
description: 'I virtually participated gctf.'
image: '/gctf/logo.png'
tags: [ CTF ]
category: 'Reversing'
draft: false 
---

## Google CTF Virtual Participation

6월 22일 오전 3시부터 6월 24일 오전 3시까지 Google CTF가 진행되었다.

그 당시에는 단지 귀찮아서 참가를 안 했지만 대회를 끝나고 보니 리버싱 문제의 수준이 높았던 것 같다.

그 이유는 모든 문제에서 솔버가 고르게 나왔지만 솔버 수가 많지 않았던 게 이유이다.

솔버는 많은 순서대로 다음과 같다.

:::note[Solvers]

+ ilovecrackmes (298pt, 20 solvers)
+ notobfuscated (298pt, 20 solvers)
+ bomberman (314pt, 17 solvers)
+ x86perm (355pt, 11 solvers)
+ arcade (363pt, 10 solvers)
+ ieee (406pt, 6 solvers)
+ rusty school (406pt, 6 solvers)

:::

이 CTF를 그저 이런 게 있었지~ 하고 넘어가기엔 솔버 수가 비정상적이게 이상적이였다. 그래서 Codeforces에서 하는 것처럼 Virtual Participation을 하기로 했다. 단, 시간 제한은 없앴다. 이 글에서는 모든 문제를 내가 시간순으로 어떻게 풀었는지 서술하겠다.

문제 풀이 시간은 '순수 풀이 시간'을 기준으로 한 것이다.

# notobfuscated (4h 10m)

우선 시작으로 이걸 하기로 결정했다. 제일 솔버가 많았기 때문이다. 나는 지금까지 솔버가 6명 (UTCTF의 In the Dark)인 문제도 풀어냈기 때문에 쉬울 줄 알았지만 오산이였다.

### 분석 1 (20m)

우선 문제는 간단히 플래그인지 아닌지 체크하는 것 같았다.

```
wane@wane:~/Hacking/ctf/googlectf/notobfuscated$ ./challenge
asdfasdufgasdfghdsyafhya
asdfasd
Incorrect, try again!
```

그렇다면 단순히 IDA를 열어서 프로그램을 보는 것이 수순이였다.

프로그램 자체는 복잡한 함수들이 나열되어있거나 하지 않았다. 바이너리도 굉장히 간단해서 3가지 파트로 나눌 수 있었다.

1. 입력을 받고 특정 규칙에 따라 파싱한다.

2. 파싱한 입력을 128비트 정수 연산을 통해 암호화한다.

3. 미리 정해진 배열과 비교하여 맞다면 플래그를 출력한다.

하지만 1번과 2번 단계가 매우 복잡했다. 1번 단계부터 뇌에서 심볼릭 익스큐션을 실행하며 봤지만 도무지 이해가 안되어서 몇 개의 입력값을 넣고 보니 답을 찾을 수 있었다.

바로 **Hex String to Bytes**이였다. 이는 Python의 `bytes.fromhex()` 메소드와 일맥상통하는 루틴이였다. 문제 풀이 방향을 확보하고 이것을 확정하는데 20분이 소요되었다.

Input을 받으면 저장하게 되는데 4비트씩 각각 13, 5, 4로 나눈 나머지를 저장했고, 이들끼리 서로소이기 때문에 $n<260$ 에서 일대일 대응 함수이다. 이렇게 암호화된 숫자를 원본 숫자 $n$ 에 대하여 $n.k$ 라고 하겠다. 그리고 이를 복호화한 값을 $n.k.d$ 라고 하겠다. 즉, $n.k.d = n$ 이다.

### 분석 2 + Excode (3h 50m)

문제는 2번 루틴이였다. 암호화 과정 특성상 하나라도 역연산을 하지 않으면 망하고 꽤나 복잡한 수학적 개념을 요구하는 문제가 다반사라 약간 뭔지도 모르고 그대로 구현했다가 큰 화를 입을 수 있었다.

분석을 하면서 가장 중요한 점은 **절대 xmmword를 숫자로 취급하지 말라**이다. xmmword는 사람이 쓰는 경우는 별로 없으며 보통 컴파일러가 최적화할 때 쓰인다.

우선 함수 `sub_555555555F50` 를 보면 다양한 힌트를 얻을 수 있었다.

```c
result = input_array;
  for ( i = 6LL; i != 38; i += 8LL )            // solve: brute forcting
  {
    v4 = ((*(_WORD *)(input_array + i - 6) >> 4) & 0xF) + ((*(_WORD *)(box_array + i - 6) >> 4) & 0xF);
    v5 = (unsigned __int16)(20 * ((*(_WORD *)(input_array + i - 6) & 0xF) + (*(_WORD *)(box_array + i - 6) & 0xF))) >> 8;
    *(_WORD *)(input_array + i - 6) = (unsigned __int8)((*(_BYTE *)(input_array + i - 6) & 0xF)
                                                      + (*(_BYTE *)(box_array + i - 6) & 0xF)
                                                      - (v5 | (12 * v5))) | (unsigned __int16)(16
                                                                                             * ((unsigned __int8)(v4 - 5 * ((unsigned __int16)(52 * v4) >> 8)) | (16 * (HIBYTE(*(_WORD *)(input_array + i - 6)) + HIBYTE(*(_WORD *)(box_array + i - 6)))) & 0x30));
    v6 = ((*(_WORD *)(input_array + i - 4) >> 4) & 0xF) + ((*(_WORD *)(box_array + i - 4) >> 4) & 0xF);
    v7 = (unsigned __int16)(20 * ((*(_WORD *)(input_array + i - 4) & 0xF) + (*(_WORD *)(box_array + i - 4) & 0xF))) >> 8;
    *(_WORD *)(input_array + i - 4) = (unsigned __int8)((*(_BYTE *)(input_array + i - 4) & 0xF)
                                                      + (*(_BYTE *)(box_array + i - 4) & 0xF)
                                                      - (v7 | (12 * v7))) | (unsigned __int16)(16
                                                                                             * ((unsigned __int8)(v6 - 5 * ((unsigned __int16)(52 * v6) >> 8)) | (16 * (HIBYTE(*(_WORD *)(input_array + i - 4)) + HIBYTE(*(_WORD *)(box_array + i - 4)))) & 0x30));
    v8 = ((*(_WORD *)(input_array + i - 2) >> 4) & 0xF) + ((*(_WORD *)(box_array + i - 2) >> 4) & 0xF);
    v9 = (unsigned __int16)(20 * ((*(_WORD *)(input_array + i - 2) & 0xF) + (*(_WORD *)(box_array + i - 2) & 0xF))) >> 8;
    *(_WORD *)(input_array + i - 2) = (unsigned __int8)((*(_BYTE *)(input_array + i - 2) & 0xF)
                                                      + (*(_BYTE *)(box_array + i - 2) & 0xF)
                                                      - (v9 | (12 * v9))) | (unsigned __int16)(16
                                                                                             * ((unsigned __int8)(v8 - 5 * ((unsigned __int16)(52 * v8) >> 8)) | (16 * (HIBYTE(*(_WORD *)(input_array + i - 2)) + HIBYTE(*(_WORD *)(box_array + i - 2)))) & 0x30));
    v10 = ((*(_WORD *)(input_array + i) >> 4) & 0xF) + ((*(_WORD *)(box_array + i) >> 4) & 0xF);
    v11 = (unsigned __int16)(20 * ((*(_WORD *)(input_array + i) & 0xF) + (*(_WORD *)(box_array + i) & 0xF))) >> 8;
    *(_WORD *)(input_array + i) = (unsigned __int8)((*(_BYTE *)(input_array + i) & 0xF)
                                                  + (*(_BYTE *)(box_array + i) & 0xF)
                                                  - (v11 | (12 * v11))) | (unsigned __int16)(16
                                                                                           * ((unsigned __int8)(v10 - 5 * ((unsigned __int16)(52 * v10) >> 8)) | (16 * (HIBYTE(*(_WORD *)(input_array + i)) + HIBYTE(*(_WORD *)(box_array + i)))) & 0x30));
  }
```

이 코드는 복잡해보이지만 분석해보면 `input_array` 와 `box_array` 의 덧셈이다. 이때 덧셈은 $\text{input\_array[i]}.k.d + \text{box\_array[i]}.k.d$ 를 수행한다, 즉, 복호화를 하고 덧셈을 한다. 이때 260 이상이 된다면 $\text{mod 260}$ 을 실행한다. 즉, 이 함수는 $ZZ(260)$ 내에서 덧셈 연산을 하는 코드이다.

**주목할 점은 사람 보통 이렇게 코드를 짜지 않는다는 것이다.** `for i in range(16):` 으로 작성 가능한 코드를 문제의 제작자는 `for i in range(4): for j in range(4):` 와 같이 작성했다. 이것은 `input_array`와 `box_array`가 1x16이 아닌 4x4 의 2차원 배열이라는 것을 암시한다.

다음 루틴은 너무 거대하다. 이때 128비트 정수 연산을 하게 된다. **절대 다 읽을 필요가 없다.** 마지막 부분에 다음과 같은 루틴이 보인다.

```c
*((_QWORD *)&storage + iterator) = _mm_or_si128(
                                         _mm_slli_epi16(
                                           _mm_or_si128(
                                             _mm_unpacklo_epi8(
                                               _mm_or_si128(
                                                 _mm_andnot_si128(v83, v77),
                                                 _mm_and_si128(_mm_add_epi8(v77, v55), v83)),
                                               (__m128i)0LL),
                                             _mm_and_si128(
                                               _mm_add_epi16(
                                                 _mm_add_epi16(
                                                   _mm_add_epi16(
                                                     _mm_srli_epi16((__m128i)storage_val, 4u),
                                                     _mm_mullo_epi16(
                                                       _mm_shufflelo_epi16(
                                                         _mm_cvtsi32_si128((our_input_f.m128i_u16[4 * iterator] >> 4) & 0x30),
                                                         0),
                                                       (__m128i)xmmword_5555555570F0)),
                                                   _mm_add_epi16(
                                                     _mm_mullo_epi16(
                                                       _mm_shufflelo_epi16(
                                                         _mm_cvtsi32_si128((our_input_f.m128i_u16[4 * iterator + 2] >> 4) & 0x30),
                                                         0),
                                                       (__m128i)xmmword_555555557190),
                                                     _mm_mullo_epi16(
                                                       _mm_shufflelo_epi16(
                                                         _mm_cvtsi32_si128((our_input_f.m128i_u16[4 * iterator + 1] >> 4) & 0x30),
                                                         0),
                                                       (__m128i)xmmword_555555557140))),
                                                 _mm_mullo_epi16(
                                                   _mm_shufflelo_epi16(
                                                     _mm_cvtsi32_si128((our_input_f.m128i_u16[4 * iterator + 3] >> 4) & 0x30),
                                                     0),
                                                   (__m128i)xmmword_5555555571C0)),
                                               (__m128i)xmmword_5555555571D0)),
                                           4u),
                                         _mm_unpacklo_epi8(
                                           _mm_or_si128(
                                             _mm_andnot_si128(v84, v82),
                                             _mm_and_si128(_mm_add_epi8(v82, (__m128i)xmmword_555555557160), v84)),
                                           (__m128i)0LL)).m128i_u64[0];
```

이게 다른 것보다 **중요한 것** 같아서 보니, `storage[i][j]` 에 영향을 끼치는 것은 `input[*][j]` 였고 다양한 상수들이 영향을 미치는 것을 확인했다. 곱셈 연산이 자주 사용되는 점, 결국 일대일대응이 되어야 한다는 점을 바탕으로 "행렬곱이 아닐까"하는 추측을 하게 되었고, 항등행렬 $I$ 를 넣어 곱셈에 사용되는 수로 추정되는 행렬을 구한 다음 몇 가지 테스트를 하니 정확했다. 마지막 루틴은 다음과 같았다.

```c
for ( j = 0LL; j != 16; ++j )
    *((_WORD *)&v100 + j) = word_55555555737C[1040
                                            * (unsigned __int8)(5 * (*((_WORD *)v110 + j) & 0xF)
                                                              + ((unsigned __int8)*((_WORD *)v110 + j) >> 4))
                                            + 260 * (HIBYTE(*((unsigned __int16 *)v110 + j)) & 0xF)
                                            + 4
                                            * (unsigned __int8)(5 * (*((_WORD *)v111 + j) & 0xF)
                                                              + ((unsigned __int8)*((_WORD *)v111 + j) >> 4))
                                            + (HIBYTE(*((unsigned __int16 *)v111 + j)) & 0xF)];
```

이건 도무지 뭘 하는지 몰랐지만 결국 한 요소가 한 요소에 대응되므로 브루트 포싱을 돌려서 얻을 수 있었다. 중요한 점은 얘가 일대일대응이 아니라는 점이였다. 하지만 2가지밖에 경우의 수가 없어서 이는 문제가 되지 않았다.

즉, 이 프로그램은 $ZZ(260)$ 내에서 행렬합과 행렬곱, 그리고 알수없는 루틴을 수행하는 것이였다.

이를 토대로 익스코드를 작성했고 플래그를 얻어냈다.

```py
from pwn import u32
from sage.all import crt
from sympy import Matrix
import numpy as np

with open('./challenge', 'rb') as f:
    raw = f.read()

bytes_data = raw[0x337c:0x337c+0x21020]
data_as_list = [ u32(bytes_data[i:i+2] + b'\x00\x00') for i in range(0, len(bytes_data), 2) ]

initial = [
    838, 1, 533, 76, 324, 68, 268, 0x301, 293, 48, 777, 796, 66, 808, 259, 818
]

last_box = [
    0x1a, 0x12c, 0x24c, 0x11c, 0x12a, 0x137, 0x210, 0x111,
    0x242, 0x31b, 0x126, 0x327, 0x329, 0x20b, 3, 0x220
]

"""
49 00 24 03 07 00 22 00  4B 02 10 02 23 01 17 00
2B 00 29 03 21 00 03 00  41 02 44 03 30 03 2A 03
"""
first_box = [
    0x49, 0x324, 0x7, 0x22, 0x24b, 0x210, 0x123, 0x17, 0x2b, 0x329, 0x21, 0x3, 0x241, 0x344, 0x330, 0x32a
]

# found by inserting identity matrix
"""
30 00 03 00 23 03 46 03  38 02 37 03 1B 03 45 02
29 01 22 03 08 00 30 03  30 01 2B 00 1C 01 22 02
"""
second_box = [
    0x30, 0x3, 0x323, 0x346, 0x238, 0x337, 0x31b, 0x245, 0x129, 0x322, 0x08, 0x330, 0x130, 0x2b, 0x11c, 0x222
]

"""
08 03 27 02 33 01 01 00  20 01 39 01 05 02 00 01
45 01 21 02 29 00 1A 01  18 03 13 00 1A 01 16 00
"""
third_box = [
    0x308, 0x227, 0x133, 0x1, 0x120, 0x139, 0x205, 0x100,
    0x145, 0x221, 0x29, 0x11a, 0x318, 0x13, 0x11a, 0x16
]

def decrypt_crt(crypted):

    lsb = crypted & 0xf # ?? % 13 = lsb
    ssb = (crypted >> 4) & 0xf # ?? % 5 = ssb
    hsb = (crypted >> 8) & 0xf # ?? % 4 = hsb

    return crt([lsb, ssb, hsb], [13, 5, 4])

def encrypt_crt(dec):

    lsb = dec % 13
    ssb = dec % 5
    hsb = dec % 4

    return (hsb << 8) + (ssb << 4) + lsb

assert len(initial) == 16

def all_possible():
    ret = []
    for i in range(4):
        for j in range(5):
            for k in range(13):
                candi = (i << 8) + (j << 4) + k
                ret.append(candi)
    return ret

ap = all_possible()

def decrypt_last_stage(crypted, box):

    last_list = [ [] for _ in range(16) ]

    for i in range(16):
        for j in ap:
            idx = 1040 * (5 * (j & 0xf) + ((j >> 4) & 0xf)) \
                + 260 * ((j >> 8) & 0xf) \
                + 4 * (5 * (box[i] & 0xf) + ((box[i] >> 4) & 0xf)) \
                + ((box[i] >> 8) & 0xf)
            #print(idx)
            res = data_as_list[idx]
            if res == crypted[i]: last_list[i].append(j)

    ret = []

    #print(last_list)
    for i in range(16):
        # it is not bifunctional I think in ZZ(260)
        # assert len(last_list[i]) == 1
        ret.append(last_list[i][0])
        #ret.append(last_list[i][-1]) # This should be work but cannot decrypt the flag

    return ret

def l2sm(l):
    ret = [[] for _ in range(4)]
    for i in range(4):
        for j in range(4):
            ret[i].append(l[i * 4 + j])
    return np.array(ret)

def sm2l(sm):
    ret = []
    for i in range(4):
        for j in range(4):
            ret.append(sm[i][j])
    return ret

FIELD = 260

def matrix_division(c, b):

    c_w = [ decrypt_crt(i) for i in c ]
    b_w = [ decrypt_crt(i) for i in b ]

    b_inv_m = np.array(Matrix(l2sm(b_w) % FIELD).inv_mod(FIELD)).astype(int)
    c_m = l2sm(c_w) % FIELD

    dec = np.dot(c_m, b_inv_m) % FIELD

    return sm2l(dec)

def matrix_sub(c, b):

    c_w = [ decrypt_crt(i) for i in c ]
    b_w = [ decrypt_crt(i) for i in b ]

    c_m = l2sm(c_w) % FIELD
    b_m = l2sm(b_w) % FIELD

    return sm2l((c_m - b_m) % FIELD)

print([hex(i) for i in initial])

initial = decrypt_last_stage(initial, last_box)

idmat = [
    1, 0, 0, 0,
    0, 1, 0, 0,
    0, 0, 1, 0,
    0, 0, 0, 1
]

# check validity
#idmatconv = matrix_sub([encrypt_crt(i) for i in idmat], first_box)
#print(bytes(idmatconv).hex())

#assert matrix_division(second_box, second_box) == idmat

#print([hex(i) for i in initial])

initial = [encrypt_crt(i) for i in matrix_sub(initial, third_box)]

#print("Target:",[hex(i) for i in initial])

initial = [encrypt_crt(i) for i in matrix_division(initial, second_box)]

#print([hex(encrypt_crt(j)) for j in sm2l((l2sm([decrypt_crt(i) for i in initial]) * l2sm([decrypt_crt(i) for i in second_box])) % 260) ])

#print([hex(i) for i in initial])

initial = matrix_sub(initial, first_box)

#print([ hex(encrypt_crt(i)) for i in initial ])

print(bytes(initial).hex())
```

```
wane@wane:~/Hacking/ctf/googlectf/notobfuscated$ ./challenge 
fcedd5ab42f188b49760fca0f99affc9
Correct! Decrypting flag
Flag: ctf{I_pr0mize_its_jUsT_mAtriCeS}
```

### 총평

어려웠다. 20명이 푼 문제였지만 만만치 않았다.

# ilovecrackmes (3h 20m)

다음으로 솔버가 많은 ilovecrackmes를 선택하여 우선 프로그램을 실행해보았다.

```
wane@wane:~/Hacking/ctf/googlectf/ilovecrackmes$ ./chal
AAB482FC7CFF14D35653BD32442895BE9F7444B19F50F658D6E313888923BEF35BC5E6AA121095AD85A8CA33CFBBC34E1DD608401A5328C252D850E5C32D574385A1D7F3B27CC672F527173E5DFC1BA273B905A8DCC6C0421AE9B7B440BBACD5DBB18AE9ABF28147FAC5F01906A335DAFDBF94686A66A5CB390D1A9177BF907E342C91A884E4A40939CB011D18A4643C9BCD61E9429406A3D8DD106113E3BBE273762B90875D9F6012686A509AEBB77847467E8F0027093CF9D9F85D1C1A804B3A35D430DE506CFA4B8A4EB23470436388BED416B17E6897EB0B0CC8EF181997ADCEDE24E31117E3055D40E7DB51AA6F08EB573254CE5FFBC46C9D8982C4172A:2961548795B067B217CD833F6A72A69625EC6FF50587A615BA9192512E3C08117B885856376259BDBA514FCF3A5684E035A60A94E218EB371D5E92BB80BDECBBC78CDCF769941366D3B9478239513CABA69797734904860D01B812A82485F9B37DD1ECBB2B399F4A097692CA970F4190277F39180020844C262A81CAF1FE2410D53216680C71DB2C1894B3E9508C14345F5EF3DB854C0BCBC8B1E4398DAC4B4FB3F80C55BAE1AC89314BE0201D3916EBB6D7742523D7D0080FC9C52C86F4AA4F9BC8AE4C901FDA00FDC982C46EBBB472CEED9F96A383C42A9937B861D9D1F66179C8D74E215747A9CDC7EE93691582D3B88CAA0E40530638F00702F04D1B862BE0CE1E7264C0CEE8D206F2F2F3C548DB123F4E6E99C791CB11AC7D01A2477C3CE47A5AE9AA65AC8BDC7981B662B5C3F9072470B9184C3085DBA13962F169695AD24FF3DCEAB8454A363857637E4BEC7023598C343A494F1C9A75A13ABD6A0A00BF76ADD82D8D2F894AD31848E4A60561289CCBCA20315CB894BCCB2D4A9E960ECDBC15C87772D003F4FA20297F4CFE6947E899F44934018A18DC4CB61E9A5911FC2BF3795984BE2AC6D0493AD761E16E32D74096FCE2022F6698C5FF13CF6C053DEF28C0AC4A57670C8783A404F71708B725ECB16E8A027A4D97E491370486202DE4AA3EF5525BB97D504C6C1F4B63098A52B0466A22D374999A5C10A49AE340

> asdfasdf
```

*"젠장, 또 암호학이야?"*

난 이런 문제가 참 싫다. 하지만 어쩌겠나? 바로 IDA를 켰다.

### 분석 1 (2h 20m)

```c
__int64 __fastcall main(int a1, char **a2, char **a3)
{
  __int64 v4[8]; // [rsp+0h] [rbp-40h] BYREF

  if ( (unsigned __int8)sub_5555555C263C(v4, 1024) == 1 )
    return (unsigned __int8)sub_5555555C224C(v4);
  std::operator<<<std::char_traits<char>>(&std::cout, "Failed to initialize\n");
  return 0xFFFFFFFFLL;
}
```

`sub_5555555C263C` 에서 일종의 Initialization을 수행한다는 것을 바로 알 수 있기에 저 함수로 들어갔다.

언뜻 보기에 간단해 보이지만 속에는 엄청난 루틴을 내포한 코드가 나를 반겼다.

하지만 몇 개 루틴을 조사하게 되면 `crypto/bn/...` 과 같은 것이 있어 인터넷에 검색해보았더니 OpenSSL의 코드가 나왔다.

OpenSSL의 Big Number 클래스를 사용하는 것 같았다. Big Number의 클래스를 참조해서 Big Number를 Python으로 파싱할 수 있었는데, 이에 따라 게싱으로 루틴을 추적해보았다.

이게 가능한 이유는 OpenSSL의 Big Number끼리 연산하는 것을 굳이 제작자가 오버라이딩하지 않았다는 것을 가정하고, 다음과 같은 과정을 통해서 게싱을 했다.

1. 만약 루틴 안에 루틴의 동작을 암시하는 문자열이 있다면 그것을 참조한다. 그것이 함수 내부의 함수에 있다면, Github에서 그 함수의 Reference를 타고 타고 올라가서 예측되는 함수를 찾는다.

2. 결과값으로 생성된 수의 Bit Length를 참조하여 루틴을 예측한다.

3. 대충 정해졌으면 테스트 해본다. 맞으면 다음으로 넘어간다. 틀리면 다른 것으로 가정하고 다시 한다.

이를 통해서 다음과 같은 루틴을 확보할 수 있다.

```py
from Crypto.Util.number import isPrime
from math import lcm

def unpack(s: str):
    s_stripped = s.replace(' ', '').replace('\n', '').lower()
    s_stripped = bytes.fromhex(s_stripped)
    return int.from_bytes(s_stripped, byteorder='little')

p = ...
p = unpack(p)


q = ...
q = unpack(q)

assert isPrime(p) and isPrime(q)

n = p * q

n_test = ...
n_test = unpack(n_test)

assert n_test == n

n_square = ...
n_square = unpack(n_square)

assert n_square == n ** 2

ps = ...
qs = ...
ps = unpack(ps)
qs = unpack(qs)

assert ps == p - 1
assert qs == q - 1

phi = ...
phi = unpack(phi)

assert lcm(ps, qs) == phi
```

일단 이 정보로 ChatGPT를 돌렸다.

<img src="/gctf/gctf1.png">

라고 한다. 실제로 이 다음에 g = N + 1를 계산하고 있었다! 확인해본 결과 `sub_5555555C263C` 함수는 Paillier 암호화 방식과 정확히 일치하는 것을 검증할 수 있었다.

### 분석 2 (30m)

이제 Initialize에서 구조체를 반환하는데 이 구조체는 Paillier 암호화의 정보에 관한 것을 반환한다.

:::note[Returns]

0. $g$
1. Big Number CTX Pool
2. $n$
3. $n^2$
4. $\lambda$
5. $\mu$
6. Unknown
7. Unknown

:::

이를 토대로 Main 루틴을 분석하는 것은 쉽다. 대충 이해한 바로는 다음과 같이 돌아가고 있었다. (Pseudocode)

```py
from phe import paillier

public, priv = paillier.generate_paillier_keypair(n_length=2048)

from random import randint

m = randint(1, 0x100000000)

print(hex(public.g)[2:].upper(), hex(public.encrypt(m).ciphertext())[2:].upper(), sep=':')
```

### 생각 & Excode (30m)

우리가 제출해야 하는 것은 Encrypted Text이고, 프로그램에서 이 텍스트를 복호화한다. 여기서 Decrypted Text가 다음과 같은 조건을 만족하면 플래그를 준다.

1. 주어진 Random Number가 포함되어야 한다.

2. `ilovecrackmes` 가 문자열에 포함되어야 한다.

Paillier 암호 시스템은 [**동형 암호**](https://en.wikipedia.org/wiki/Homomorphic_encryption)이다! 그냥 주어진 암호화된 문자열 + `ilovecrackmes\x00\x00\x00\x00` 를 계산한 뒤 넘겨주면 된다!



```py
from phe import paillier
from pwn import *
from Crypto.Util.number import bytes_to_long, long_to_bytes

context.log_level = 'debug'

#test_public, test_private = paillier.generate_paillier_keypair(n_length=2048)

m = randint(1, 0x100000000)

#p = process('./chal')
p = remote("ilovecrackmes.2024.ctfcompetition.com", 1337)
_ = p.recv() # for proof-of-work
received = p.recvline().split(b':')
g = int(received[0], 16)

public = paillier.PaillierPublicKey(g - 1)
c = paillier.EncryptedNumber(public, int(received[1], 16))

"""
g = test_public.g
c = paillier.EncryptedNumber(test_public, test_public.encrypt(m).ciphertext())
"""

payload = public.encrypt(bytes_to_long(b'ilovecrackmes\x00\x00\x00\x00'))
payload += c

p.sendline(long_to_bytes(payload.ciphertext()).hex().encode())
p.interactive()
```



```
wane@wane:~/Hacking/ctf/googlectf/ilovecrackmes$ python -u "/home/wane/Hacking/ctf/googlectf/ilovecrackmes/akali.py"
[+] Opening connection to ilovecrackmes.2024.ctfcompetition.com on port 1337: Done
[DEBUG] Received 0x1e bytes:
    b'== proof-of-work: disabled ==\n'
[DEBUG] Received 0x580 bytes:
    b'C0DEA3989271831C360CF0364E644756AF6AFD7376CAF0E3E70AA7B52477F468B5E2ED9669E9BE8C5F72751236EA3C87559E3D1C5E5F9EF130B0D0ADDD90D509D718FA12FDA8D0217F0F6D8A738B713357991DD9144036B1E3B5DE9E73552AE5483498CCB637B757C297916164D5F32999CDA2B292BD2508B02D6D00CE8F9F1BAE5E61B7470A71B91670B399FEFF131812BFB7AB4DD58EDA774085A3CAADDF2D3507E85F02CFA448109BA28B5B4A4232332C068D0FE51CA17EC4C4283573AE5DFB4C844C9AB9EB56DCCF858C190CC151C321EFE6C9AE0914739206E4DD9380EDEF87EA74D33CEB72AD6EA801CC5FE41FAFF64B5E947A8F44E2492D7D42B2A03C:8D1D66385172BB867573559897A2B91245C4B2AF4BF318271912F48F2FEDF9A5BE85D620D4E8E62B761587B5F49D8D45F5A6D562139B9069B58B3738F9F8CBE889C70CEA9B04C345394925072DD2F47A859F339FA754BF05A0712A584B03F7C537F520D17497DD17711238AE86D99B8CA2F4B882D35A03E8F40A80CD8F3E4B75EF7D67F4CEA570DC4A29457EFBCE744AD791B5CCB797F602BFE7F7E12D92D2034BD1A519494558CF68F89DEC084870DEC6E7C954773633D8C3729937039A066C874FBBB61A681A9E843B602491FFDC363838BD9894EE7F1EF59EA1BF4B2270856E80004572567D618213277B0E5C41D546F9376740A94919C861BE961F2FB3EC73840FEBF15B64B285D07D6FA24755A103F29670B163EA676DA894C5720F1C372AF24146CEF8C0D4C65D03721646C8AA85828FE9AAB4518B8549197E3D5FCFEE732B8357C58AE4474CBC9C71B815FA6F809EE77DC690A2F4DA1D080F4A655874726940FEBEEAEEF559DA6DFAD0CD5ECA69BE92813690E9926001A06444FC6E367F25E021590097E092F49D286C0DA47D50499DD2BB3AC1AF0D5BDE3CAA6481C[DEBUG] Received 0x85 bytes:
    b'1E32A6CD6170517CB02D3A91406BA0828EE97E46C01F80F90664EF19BA2A802853EE8EC4228E13CEB1ED70C8C08EC896A9C18308B2D2C9CE9CC3D669E16335BE8\n'
[DEBUG] Received 0x85 bytes:
    b'1E32A6CD6170517CB02D3A91406BA0828EE97E46C01F80F90664EF19BA2A802853EE8EC4228E13CEB1ED70C8C08EC896A9C18308B2D2C9CE9CC3D669E16335BE8\n'
    b'\n'
    b'> '
[DEBUG] Sent 0x401 bytes:
    b'0bcba23c5498cd8b6ce0882f878cad1c48e9ec17e98173301e9973ccf3b5478800550f06513c4821a10ff8996d72ac8f70a72c319990949be2b72880374c157fdd852544e9d23a0450ef6bead8dea9dc1cdc4a13f8ba2c243e58b2b93e502a41fee0b51818ea0b1f6e0b26128468a503793daba12243be4f2700e4dacf23ea7f054b1159fd9ec48244f5521a7628bde12d343c7595113f158a723199c0bcb1ec8ab9106d779524f8c3a67a8cbcf32e41674977b634d3a469f760fcf05226034db439f5cf7784573e9ac3311b7a7757736509bed57630aee14bd146cbe374d9ece2f735b1b56e456a94750fc876bcf6e3a7ca4333dfb3d99188a8811a0780dee1d3543a92b1a38e7998202a1060b5fb258f34240642c16ffe7338680ce59574eff5d388fc6f496ab3b747b382f8e461678aa6830655564b5fbfc1644eb542cd6dd6fb73ce924abf4b7e29ca9aeadc5826b21399b78bbee889b09e7034b3e72f44eaa6599ae64d32a4e14ea553c77ffb168cc268db3265dfa25418aa787d2d2d2207fda6f7e6b6981b2cd41137ddcdef12db5e0080081ff3839605eabe9a0145c3008dd09e089943e644dba65e882cfb06e7d61714cd8c9bd8c24c6b8a50e78444f8c2fb55c4951ba78d5e317e1af480efb162da5eb8a486af7e61bf17a7e9b47fdacc39dba8d3aadbd3811b3c1211c0f33c44be57d3aeb1f1c84870a61568d2fd\n'
[*] Switching to interactive mode

> [DEBUG] Received 0x27 bytes:
    b'CTF{4r17hm371c_w17h0u7_d3cryp7i0n_f7w}\n'
CTF{4r17hm371c_w17h0u7_d3cryp7i0n_f7w}
```



### 총평

처음부터 끝까지 게싱이였다. notobfuscated보다 쉬웠다.



## 비고

풀 때마다 여기에 업데이트 하겠다.
