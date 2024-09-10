---
title: CTF Reversing Cheat Lists - Part 1
published: 2024-09-10
description: 'CTF reversing guide for "math reversing" problems'
image: '/ctfrv1/image.png'
tags: [ CTF ]
category: 'Reversing'
draft: false 
---

# CTF Reversing Cheat Lists - Part 1

리버싱을 하다보면 어찌보면 단순(?)하게 플래그를 얻어내는 경우가 있지만 이와 반대로 실행 루틴을 발견해냈으나 어떻게 복호화할 지 몰라 허둥대는 경우가 종종 있다.

난 이 경우를 지극히 싫어한다. 만약 이런 문제가 CTF 때 나온다면 필히 출제자를 욕하리라.

하지만 그렇게 싫어하고만 있다면 그저 한 Math-Rev Hater가 될 뿐이다. 풀지도 못하고 욕하는 것보다 풀고 욕하는 게 훨 낫다. 따라서 여러분들을 위해 내가 몇 가지 응용되는 테크닉을 알려줄 것이다.

## Linear Equation (⭐⭐⭐⭐⭐)

연립방정식은 진짜, 굉장히 많이 나온다. 중요한 점은 체 $F$ 내의 $n$개의 변수 $x_1, x_2, x_3, ..., x_n$이 존재하고 이에 따른 $F$ 내의 **선형식** $n$개: $P_1(x)=p_1, P_2(x)=p_2, ..., P_n(x)=p_n$가 있다면 변수들을 **완벽히** 복구할 수 있다.

말이 어려운데, 변수 $n$개가 있다면 방정식 $n$개가 있을 때 비로소 유일하게 해를 도출할 수 있다.

가우스 소거법을 이용해서 해를 구할 수 있다. Sage에서 계수 행렬을 정의하고, 값 벡터를 정의한 뒤 `solve_right` 함수를 통해 구할 수 있다.

```py
# x + 2y + 3z = 4
# 4x + 8y + 11z = 18
# 2x + y - z = 2
M = matrix(RR, [
    [1, 2, 3],
    [4, 8, 11],
    [2, 1, -1]
])
V = vector(RR, [
    4,
    18,
    2
])

solution = M.solve_right(V)
```

```bash
(-3.33333333333333, 6.66666666666667, -2.00000000000000)
```

### Lagrange Interpolation

$$
P(x) = a_nx^n+...+a_1x+a_0
$$

과 같은 $n$차 다항식 $P(x)$에 대해 $n$개의 $P(x_0), P(x_1), ..., P(x_n)$ 결과값이 있을 때 $a_0, a_1, ..., a_n$을 구하기 위해 쓰이는 것이 라그랑주 보간법이다.

하지만 그런 거 쓸 필요 없이 $a_0, a_1, ..., a_n$에 대해 선형식이므로 연립방정식을 통해 쉽게 구할 수 있다.



:::note[연습문제]

1. [multipoint](https://dreamhack.io/wargame/challenges/7)

:::



## Polynomial Equation (⭐⭐)

고차 다항식을 인수분해하는 것은 어렵다. 포기하도록 하자. 하지만 저차 다항식의 제곱꼴의 곱으로 이루어진 고차 다항식은 중근을 구하기 쉬우므로 그럴 땐 구하도록 하자. (예: $x^4 + 4x^3+6x^2+4x+1 = (x+1)^4$, 근: $x=-1$)

문제는 계수가 유한체 안의 원소인 다항식 환에 속하는 다항식 $P(x), N(x) \in PR(GF(p))$ 에 대해 $P(x)^{e} \equiv R(x) \mod{N(x)}$ 에 대하여 $R(x), N(x)$가 주어질 때 $P(x)$를 구하는 것은 어렵다. 이는 Polynomial RSA 꼴이다.

하지만 $N(x)$의 최고차항이 충분히 낮아 인수분해할 수 있게 된다면 $P(x)$를 구할 수 있다.

Write-up wrote by [Yorix](https://dreamhack.io/wargame/writeups/18576) (multipoint2):

```py
PRIME = 251
P.<x> = PolynomialRing(GF(PRIME))
payload = x^38

# We can know that n is ax^38 + bx^37 + ...

r_coeffs = [ 0x90, 0x58, 0x53, 0x3B, 0x7A, 0x33, 0x65, 0x92, 0x32, 0x9D, 0x29, 0x6B, 0x8E, 0xE0, 0x0C, 0x7F,
    0x5F, 0xD1, 0x95, 0xC2, 0xA2, 0xF9, 0x21, 0x84, 0x90, 0xEA, 0x09, 0xBB, 0x08, 0x78, 0x5F, 0xA7,
    0xDD, 0x85, 0xBD, 0x86, 0xEC, 0x9B ]

r = 0
for i, c in enumerate(r_coeffs):
    r += c * (x^i)

n = payload - r

p, q = n.factor()
p = p[0]
q = q[0]
e = 65537
phi = (PRIME**p.degree() - 1) * (PRIME**q.degree() - 1)
d = pow(e, -1, phi)

cipher_coeffs = [ 0x6b, 0x82, 0x5a, 0xa3, 0x08, 0x9f, 0xb7, 0xe2, 0x06, 0x4a, 0xe7, 0xa7, 0x0e, 0x3d, 0x68, 0x56, 0x94, 0x4d, 0x37, 0x30, 0x70, 0x9b, 0x43, 0x98, 0x86, 0xb8, 0x73, 0xb8, 0x83, 0x14, 0x21, 0x38, 0x5a, 0x94, 0xcd, 0x9e, 0x29, 0x77 ]

cipher = 0
for i, c in enumerate(cipher_coeffs):
    cipher += c * (x^i)

plain = pow(cipher, d, n)

print(''.join([chr(i) for i in pow(cipher, d, n).coefficients()]))


```



:::note[연습문제]

1. [multipoint2](https://dreamhack.io/wargame/challenges/1033)

:::



## Affine Transform (⭐⭐⭐)

어파인 변환 개념은 조금 어려운 문제에서 나온다. 어파인 변환은 벡터 $M, B$, 행렬 $A$, 에 대해

$$
P = AM+ B
$$

를 수행하고 $P$를 구하는 것이다. 이때  $P$는 $M$과 같은 공간 내에 있다. $P$에 대해 같은 $A, B$에 대해 어파인 변환을 수행한 벡터를 $P'$이라 하면 $P'$과 $M$은 같은 공간 내에 있다. 증명은 삼단논법이다.

문제는 [S-box](https://en.wikipedia.org/wiki/S-box)과 관련되어 나온다. S-box는 원래 수식으로 나타낸다면 2차 이상의 다항식이 나온다. 이때 S-box를 여러 번 수행하면 고차 다항식이 나와서 위에서 말했듯이 원본 값을 구하기 어려워진다. 하지만 S-box가 일종의 어파인 변환이라면 위의 정리로 인해 아무리 수행해도 1차 다항식이 나온다.

S-box가 어파인 변환인지 확인하는 것은 다음과 같이 이루어질 수 있다.

```py
from arybo.lib import *
from arybo.tools import *

mba4 = MBA(4)

def main():
    sbox = [4, 7, 2, 1, 8, 11, 14, 13, 15, 12, 9, 10, 3, 0, 5, 6]
    E,X = mba4.permut2expr(sbox)
    print(E.vectorial_decomp([X]))

if __name__ == "__main__":
    main()
```

```py
App NL = Vec([
0,
0,
0,
0
])

AffApp matrix = Mat([
[1, 0, 0, 1]
[1, 1, 0, 1]
[0, 1, 1, 0]
[0, 0, 1, 1]
])

AffApp cst = Vec([
0,
0,
1,
0
])
```

`NL`, `matrix`, `cst`의 원소가 모두 정의한 체 (보통 $GF(2)$) 내의 원소라면 이는 어파인 변환으로 묘사할 수 있다는 것이다.

:::note[연습문제]

1. [Shadow of Encryption](https://dreamhack.io/wargame/challenges/1302)
2. LACUCARA VM - Codegate 2024 Quals

:::

## Bit Operation as GF(2) (⭐⭐⭐⭐)

암호화 과정을 볼 때 `<<`나 `&`, `|` 등 다양한 비트 오퍼레이션을 볼 수 있고 보통 이럴 경우 머리가 복잡해진다.

이때 사람들은 무지성으로 [z3-solver](https://github.com/Z3Prover/z3)를 이용해 풀곤 한다. 나도 그렇다. 왜냐하면 보통 풀리기 때문이다. 하지만 z3는 기본적으로 SAT Solver이다. 식의 길이가 커지면 z3가 자동으로 식을 축소하지 않기 때문에 푸는데 시간이 걸린다.

암호화는 거의 바이트 단위로 이루어지는데 이걸 8개의 비트로 다시 재정의하자. 비트는 $GF(2)$ 내의 원소이다. 여기서 $GF(2)$ 내의 원소의 특징을 알고 각 비트에 대해 식을 구성하게 되면 식이 간단해지므로 무지성으로 긴 식을 때려넣는 것보다 효율적일 수 있다.

1. `&` 연산은 $GF(2)$ 내에서 `*` 연산과 동치이다.

2. `|` 연산은 $GF(2)$ 내에서 `+` 연산과 동치이다.

3. `<<`, `>>` 연산을 맞이했을 때는 비트 배열을 그냥 Shift하자.



비효율적이긴 하다만, 구현체이다.

```py
class GFvar:
    def __init__(self, bits : list):
        while (bits[-1] == 0 and len(bits) != 1):
            bits.pop()
        self.bits = bits

    def __rshift__(self, shift : int) -> 'GFvar':
        bits = self.bits

        while (bits[-1] == 0 and len(bits) != 1):
            bits.pop()

        if shift >= len(bits):
            return GFvar([0])

        return GFvar(bits[shift:])

    def __lshift__(self, shift : int) -> 'GFvar':
        return GFvar([0] * shift + self.bits)

    def __xor__(self, other : 'GFvar | int') -> 'GFvar':
        bits = self.bits

        if isinstance(other, GFvar):
            other = other.bits
        else:
            other = [(other >> i) & 1 for i in range(int(other).bit_length())]

        sl, ol = len(bits), len(other)

        if sl == ol:
            new_v = [(bits[i] ^^ other[i]) for i in range(sl)]

        elif sl > ol:
            new_v = [(bits[i] ^^ other[i]) for i in range(ol)] + bits[ol:]

        else: # sl < ol
            new_v = [(bits[i] ^^ other[i]) for i in range(sl)] + other[sl:]

        return GFvar(new_v)

    def __and__(self, other : 'GFvar | int') -> 'GFvar':
        bits = self.bits

        if isinstance(other, GFvar):
            other = other.bits
        else:
            other = [(other >> i) & 1 for i in range(int(other).bit_length())]

        sl, ol = len(bits), len(other)

        new_v = []
        for i in range(min(sl, ol)):
            if other[i] == 0:
                new_v.append(0)
            else:
                new_v.append(bits[i])
        
        return GFvar(new_v)

    def __repr__(self) -> str:
        return f'{self.bits}'
```



:::note[연습문제]

1. [XOR Disaster](https://dreamhack.io/wargame/challenges/1294)

2. [Red-Black Christmas Tree](https://dreamhack.io/wargame/challenges/737)

:::



# 결론

Part 2로 돌아오겠다. 다시 생각해도 이 망할 Math-rev는 질리지도 않고 나온다. 문제를 좀 열심히 만들어서 재밌는 문제를 만들자.
