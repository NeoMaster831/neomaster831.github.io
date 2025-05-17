---
title: LakeCTF Finals Write-Up
published: 2025-05-18
description: 'Solved 3 revs'
image: '/lctf-finals/lctf.png'
tags: [ CTF ]
category: 'Reversing'
draft: false 
---

Solved..
+ **Fastprocess** (116 pts)
+ **Jump Around** (116 pts)
+ **Hygiene** (140 pts)

마지막 문제가 0솔이니 나도 세계 공동 1등 리버서!? 라고 생각 한 번 해 봤다.

# Fastprocess

문제의 `f` 함수를 다음과 같이 Python pseudocode로 옮길 수 있다.

```py
def func_a(c):
    if c <= 0x1e:
        return 0
    if c == 31:
        return 1
    v7 = [0] * 0x100
    v7[31] = 1
    v2 = 1
    v4 = 0
    for _ in range(32, c + 1):
        v6 = v2
        v3 = modsub(v2, v7[v4], 2 ** 64)
        v7[v4] = v6
        v2 = modadd(v6, v3, 2 ** 64)
        v4 = (v4 + 1) & 0x1f
    return v7[(v4 + 31) & 0x1f]
```

이 경우 시간 복잡도는 `O(c)` 이며 마지막 Hard 난이도를 통과할 수 없다. 하지만 함수가 피보나치를 닮아 있으니 행렬 제곱으로 환원할 수 있다. 이때 시간 복잡도는 `O(log c)` 가 된다.

```py
def func_a_log(c: int) -> int:
    # base cases
    if c <= 30:
        return 0
    if c == 31:
        return 1
    mask = (1 << 64) - 1
    dim = 33

    M = [[0] * dim for _ in range(dim)]
    M[0][0]   = 2               # coefficient of X_n
    M[0][32]  = mask            # -1 mod 2^64
    for i in range(1, dim):
        M[i][i-1] = 1           # shifting rows
    def mat_mult(A, B):
        C = [[0] * dim for _ in range(dim)]
        for i in range(dim):
            Ai, Ci = A[i], C[i]
            for k, a in enumerate(Ai):
                if a:
                    rowB = B[k]
                    mul  = a
                    for j in range(dim):
                        Ci[j] = (Ci[j] + mul * rowB[j]) & mask
        return C

    def mat_pow(mat, exp):
        R = [[0] * dim for _ in range(dim)]
        for i in range(dim):
            R[i][i] = 1
        base = mat
        while exp > 0:
            if exp & 1:
                R = mat_mult(R, base)
            base = mat_mult(base, base)
            exp >>= 1
        return R

    P = mat_pow(M, c - 32)
    return (P[0][0] + P[0][1]) & mask
```

```bash
wane@wane:~/Hacking/ctf/lake25f/badp$ sha256sum solve1.py
74d8835780ade07f7407ad71cfe3f69a64393a4b9ab5afac8991209bb3afc1b4  solve1.py
```

퍼블 먹었다.

# Jump Around

모든 코드 조각을 쪼개놓는 난독화를 해 놓은 것을 알 수 있다. 디컴파일러를 작성하지 않아도 문제가 꽤나 정직한 AES라서 피지컬으로 극복이 된다.

```bash
wane@wane:~/Hacking/ctf/lake25f/ja$ sha256sum test.py
496dfe17ac0b82f355e4d58dd7c1e20685679e845e08390c80d677dd85a72092  test.py
```

# Hygeine

특이한 문제이다. 시스템 해킹과 접목한 문제인 것을 알 수 있다. 우선 문제에서 제공하는 암호화 함수는 다음과 같다.

```py
def crypt(plain, counter=0):
    key = b"???" # Originally random.
    nonce = b"\x00" * 8 # 고정
    cipher = ChaCha20.new(key=key, nonce=nonce)
    cipher.seek(counter * 64)
    ciphertext = cipher.encrypt(plain)
    return ciphertext
```

우리가 지정할 수 있는 것은 `counter` 와 `plaintext` 이지만, `counter` 의 범위는 3 이상이여야 한다. `counter`를 조작하여 임의의 암호문에 대해 평문을 복호화할 수 있지만 우리가 원하는 플래그는 블록 1과 2에 있으므로 우리는 원래대로라면 이 문제를 풀 수 없다.

신기한 점은 `counter`가 3일 때 모든 바이트가 0이여야 하지만 실제로 랜덤한 주소가 릭이 되고 있음을 확인하였다. 이는 스택에 정리되지 않고 잔류하는 바이트들이 정보를 형성한 것으로, 이것을 통해 문제를 풀 수 있다.

실제로 스택을 조사하면 잔류하는 키를 찾을 수 있다. 이를 통해 복호화하면 된다.

```bash
wane@wane:~/Hacking/ctf/lake25f/hyg$ sha256sum a.py
3753fc23347be4e569f1516cee29da1fc06d8a7895a3edd1e43645a4cb8a7621  a.py
```