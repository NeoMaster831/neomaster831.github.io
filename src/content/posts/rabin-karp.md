---
title: Rabin-Karp Breaker
published: 2024-06-03
description: 'Breaking Rabin-Karp with LLL'
image: ''
tags: [LLL]
category: 'Crypto'
draft: false 
---

# Rabin-Karp Breaker Wane

:::note[Credit]
Wane
:::

# 라빈-카프 알고리즘?

라빈-카프 알고리즘은 **KMP** (Knuth-Morris-Prett), **Aho-Corasick** 과 같은 문자열 매칭 알고리즘 중 하나로, **라빈-카프 해싱** 을 이용하여 문자열을 $O(N)$ 에 찾는 알고리즘이다. [Ref](https://en.wikipedia.org/wiki/Rabin%E2%80%93Karp_algorithm)

여기서 라빈-카프 알고리즘을 안 쓰는 이유는, 라빈-카프 알고리즘에 사용되는 라빈-카프 해싱을 충돌시키는 데이터 $a$, $b$ 쌍을 *아주* 효율적으로 만들 수 있기 때문이다. 따라서 이 해싱, 혹은 알고리즘을 실제에서 사용한다면 큰 화를 입을 수 있다.

# 어떻게?

**라빈-카프** 해싱은 다음과 같이 계산된다.

:::note[Rabin-Karp Hash]
소수 $p$ , 데이터 $d$ (일반적으로 문자열) 에 대해, $n$ 은 $|d|$ 이다. ($|x|$ 는 $x$ 의 길이를 말한다.) 이때, **Rabin-Karp 해시** $H(p, d)$ 는 다음과 같이 계산된다.
$$
H(p, d) = \sum_{i=1}^Nd[i]p^i
$$
:::

**Rabin-Karp 해시**의 서로 다른 데이터 $d_a$, $d_b$ 가 해시가 충돌한다는 것은 다음과 같이 표현된다.

$$
H(p, d_a) = H(p, d_b) \\
H(p, d_a) - H(p, d_b) = 0 \\
\sum_{i=1}^Nd_a[i]p^i -\sum_{i=1}^Nd_b[i]p^i = 0 \\
\sum_{i=1}^N(d_a[i] - d_b[i])p^i = 0
$$

여기서 우리는 한 가지 개념을 도입할 수 있다. 우선 유명한 문제인 **Integer Relation Problem**에 대해 알아보자.

:::note[Integer Relation Problem]
길이가 $n$ 인 **정수 집합** $A = \{ a_i\ |\ 1 \leq i \leq n \}$ 에 대해, 다음을 만족시키는 **정수 집합** $X = \{ x_i\ |\ 1 \leq i \leq n \}$ 가 존재한다면 그것을 구하라.
$$
\sum_{i=1}^n x_i a_i = 0
$$
:::

이 문제의 답 $X$ 를 $d_a[i] - d_b[i]$, $A$ 를 집합 $P = \{p^i \  | \ 0 \leq i < n \}$ 로 정의한다면 **Rabin-Karp 해시**는 **IRP**의 일종으로 받아들일 수 있다.

[**Subset Sum 문제**](https://en.wikipedia.org/wiki/Subset_sum_problem) 또한 **IRP**의 일종으로 볼 수 있기에 **IRP** 또한 NP 문제로 분류되어 풀기 정말 어려울 것 같지만 정말 놀랍게도 **IRP**는 NP 문제는 맞지만 빠르게 풀 수 있다. (???)

우리는 창의적으로 **IRP**에 대해 다음과 같은 [격자](https://en.wikipedia.org/wiki/Lattice_(order)) $\mathcal{L}$ 를 정의할 수 있다.

$$
\mathcal{L} = \begin{bmatrix}
1 & 0 & 0 & \cdots & 0 & a_1 \\
0 & 1 & 0 & \cdots & 0 & a_2 \\
0 & 0 & 1 & \cdots & 0 & a_3 \\
\ & \ & \ & \ddots & \ & \ \\
0 & 0 & 0 & \cdots & 1 & a_n
\end{bmatrix}
$$

$\mathcal{L}$ 에서 각 Row를 $\mathbf{b}_i$ 라 하면 다음과 같은 벡터 $\mathbf{v}$ 는 $\mathcal{L}$ 안에 있다.

$$
\mathbf{v} = \sum_{i=1}^nx_i\mathbf{b}_i \\
= (x_1, x_2, \cdots, x_n, a_1x_1+a_2x_2+\cdots+a_nx_n) \\
\approx (x_1, x_2, \cdots, x_n, 0)
$$

$x_i$ 는 충분히 작으므로 $\mathbf{v}$ 는 $\mathcal{L}$ 의 **Shortest-Vector** 가 된다. 그리고 보다시피 $\mathbf{v}$ 는 $X$ 의 모든 원소를 가지고 있으므로, **IRP** 문제를 $\mathcal{L}$ 에서의 **Shortest-Vector** 를 찾는 문제로 변형 가능하다.

안타깝게도 **Shortest-Vector Problem** (SVP)는 현재 NP-Hard로 분류되어 있다. 하지만 특정 알고리즘을 통해 특정 조건 내에서 SVP를 다항 시간 안에 구하는 알고리즘, [LLL](https://en.wikipedia.org/wiki/Lenstra%E2%80%93Lenstra%E2%80%93Lov%C3%A1sz_lattice_basis_reduction_algorithm)이 있다. 이것을 사용해서 대강 $n \leq 100$ 내에서 아주 빠르게 IRP의 해답을 구할 수 있고, 이를 라빈-카프 해싱에 적용할 수 있다.

# POC

Sagemath를 이용한 풀이이다. $p = 29,m=10^9+7,n=100$ 에 대해 알파벳 소문자로 된 해시 충돌 쌍을 구하는 코드이다.

```python
from sage.all import *

p = [ 29 ]#[ 29, 31, 37, 41, 43, 47, 53, 59, 61, 67 ]
c = 69
n = 100
m = 1000000007

def get_x(p, n, c, m):

    # First we denote lattice L

    L = []
    for i in range(n):
        L.append([1 if i == j else 0 for j in range(n)] + [ c * pow(j, i, m) for j in p ])
    
    L = Matrix(L)
    redux = L.LLL()
    sparce = redux[0]

    x = sparce[:n]
    values = sparce[n:]

    assert(all(abs(xi) <= 25 for xi in x))
    assert(all(vi == 0 for vi in values))

    return x

def solve():
    x = get_x(p, n, c, m)

    a = ""
    b = ""
    for i in range(n):
        if x[i] < 0:
            b += chr(ord('a') - x[i])
            a += "a"
        else:
            b += "a"
            a += chr(ord('a') + x[i])
    
    print(a, b)
    
solve()
```

후에 소개할 것이지만 $p$ 가 여러 개이고 수많은 $p$ 에 대해 충돌하는 쌍 또한 구할 수 있다.

이때 $\mathcal{L}[0]$ 이 우리가 원하는 해답이고 상수 $c$ 같은 경우 $X$ 의 모든 원소가 $\pm25$ 를 넘기지 않게 조정하기 위한 상수이다.

## Exercise

[BOJ 13318 - 위험한 해싱](https://www.acmicpc.net/problem/13318)  
2019 X-MAS CTF: Hashed Presents  

# More

$m$ 개의 $p$ 에 대해 만족하는 해시 충돌 쌍은 다음과 같은 격자를 Reduce함으로써 얻을 수 있다. 왜인지는 생각해보길 바란다.

$$
\mathcal{L} = \begin{bmatrix}
1 & 0 & 0 & \cdots & 0 & {p_1}^0 & \cdots & {p_m}^0 \\
0 & 1 & 0 & \cdots & 0 & {p_1}^1 & \cdots & {p_m}^1 \\
0 & 0 & 1 & \cdots & 0 & {p_1}^2 & \cdots & {p_m}^2 \\
\ & \ & \ & \ & \ddots & \ & \\\
0 & 0 & 0 & \cdots & 1 & {p_1}^n & \cdots & {p_m}^n
\end{bmatrix}
$$

그리고 Subset-Sum 문제 또한 특정 조건 하에 LLL으로 풀이할 수 있다. 다음과 같은 격자를 생각해보자.

$A = \{ a_i\ |\ 1 \leq i \leq n \}$ 에서 몇 개를 골라 합이 $K$ 가 되는 Subset-Sum 문제를 생각할 때,

$$
\mathcal{L} = \begin{bmatrix}
1 & 0 & 0 & \cdots & 0 & -a_1 \\
0 & 1 & 0 & \cdots & 0 & -a_2 \\
0 & 0 & 1 & \cdots & 0 & -a_3 \\
\ & \ & \ & \ddots & \ & \ \\
0 & 0 & 0 & \cdots & 1 & -a_n \\
0 & 0 & 0 & \cdots & 0 & K
\end{bmatrix}
$$

그럼 이상!!