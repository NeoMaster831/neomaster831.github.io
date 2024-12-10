---
title: LakeCTF '24-'25 Quals Write-up
published: 2024-12-10
description: 'Solved two problems :3'
image: '/lctf-writeup/lctf-logo.png'
tags: [ CTF ]
category: 'Reversing'
draft: false 
---

Solved...
+ **smol-rev** (410 pts)
+ **silent-lake** (402 pts)

두 문제 모두 리버싱이고, 상당히 어려웠다. Write-Up을 간단히 써 볼까 한다.

# smol-rev

`chal` 파일이 주어진다. 이 파일은 Base64-ZStandard로 압축된 파일이다. 왜 이렇게 주는지는 모르겠지만, 어쨌든 압축을 해제하고 프로그램을 분석하게 되면 통상적인 컴파일러로 컴파일한 것이 아니라는 것을 볼 수 있다.

VM 문제처럼 접근하여 규칙을 찾고 분석해나가면 된다. 먼저 입력을 받는 부분을 차근차근 분석하면,

> -  숫자로만 이루어져 있다.
> -  4096자를 넘기면 안 된다.
> -  끝은 무조건 `$` 로 끝나야 한다. 이는 프로그램의 End를 나타낸다.

의 규칙을 지키지 않으면 `nope :3` 을 출력하고 프로그램이 터지게 된다. 이때 입력 규칙을 파싱하는 곳을 자세히 분석하면,

```
1. `$` 앞의 문자열을 Strip 한다.
2. 다음을 반복한다.
2-1. 숫자를 파싱한다. 이는 arg1에 저장한다. 
2-2. 숫자를 파싱한다. 이는 arg2에 저장한다. 
2-3. 숫자를 파싱한다. 이는 arg3에 저장한다. 
2-4. `0x415360` + 9 * arg2 + arg1 에 해당하는 숫자를 가져온다.
2-5. 만약 -1이 아니라면, 프로그램을 터트린다.
2-6. -1이라면, 해당 숫자를 arg3으로 저장한다.
```

이때 `0x415360` 전역 배열을 보면 다음과 같다.

```
FF FF FF FF FF FF FF FF  FF FF FF FF FF FF 03 FF
08 05 FF FF 01 FF 02 FF  FF FF FF FF FF FF 05 FF
07 FF FF FF FF FF 04 FF  FF FF 01 FF FF FF 09 FF
FF FF FF FF FF FF 05 FF  FF FF FF FF FF 07 03 FF
FF 02 FF 01 FF FF FF FF  FF FF FF FF 04 FF FF FF
09
```

아직까지는 무엇을 뜻하는지 분간이 안 되지만, 차근차근 분석해나가다 보면 다음과 같이 소수 배열을 찾을 수 있었다.

<img src="/lctf-writeup/2dd.png">

솔직히 여기서 갑자기 소수 배열이 나와서 포기할 뻔 했다. 하지만 차근차근 분석해 나가다 보니, 어떤 값을 계산한 후 그 값을 `0D4C2086` 와 반복하여 비교한다. 그리고 맞는 경우, 어떤 전역 변수를 1 증가시킨다.

마지막으로 전역 변수의 값을 27과 비교하여 맞다면 파일 디스크립터 24에서 플래그를 읽어와서 출력하는 모습을 보았다.

## Key Point

중요한 점은 함수 트레이싱을 돌렸는데 어떤 함수가 243번 호출되는 것을 확인했다.

`0x415360` 전역 배열은 9x9 이고, 해당 함수에서 이 배열을 간헐적으로 참조하고 있었다. 루틴이 복잡해서 몇 번째 원소에 접근하는지는 구하지 못했다. 하지만 곧이어 치명적인 힌트를 발견할 수 있었다.

<img src="/lctf-writeup/1dd.png">

정확히 소인수가 9개이고, 앞서 구한 소수 배열과 정확히 일치하는 모습을 보았다. [이와 아주 비슷한 문제](https://dreamhack.io/wargame/challenges/968)를 풀어본 경험이 있다. 이후로부턴 그냥 끝났다. 스도쿠를 풀어서 입력 형식에 따라 답을 제출하면 풀리게 된다.

```bash
wane@wane:~/Hacking/ctf/lctf/smol-rev$ sha256sum a.cpp solve0.py
9268c59e2dcefb079e66b2c1d522a69be437070013988127788d93adb9145ac8  a.cpp
09642dee5d87b9834adbe70586e81f03684151665243697416743b65dba4e241  solve0.py
```

`EPFL{it-is-a-smol-step-for-me-but-an-even-smoler-one-for-compilers}`

# silent-lake

CodeQL 의 Precompiled Query들이 주어진다. 목표는 해당 쿼리를 만족하는 소스 코드를 작성하여 `res` 반환값이 `correct`가 되게 만들면 되는 문제이다. 이를 분석하기 위해 우선 `codeql decompile` 명령어를 사용할 수 있다.

```bash
wane@wane:~/Hacking/ctf/lctf/sillake$ codeql query decompile vscode-codeql-starter/codeql-custom-queries-cpp/example.qlx > example.qlo
wane@wane:~/Hacking/ctf/lctf/sillake$ cat example.qlo
# ...
```

이렇게 명령어를 치게 되면 약 16000줄 가량의 IL이 나오게 된다. 대충 중요한 것만 뽑는다면 약 15000번째 줄부터 마지막 줄까지이다. 천 줄 가량의 코드를 분석하는 건 그렇게 어려운 일이 아니니, 한 번 해보면 된다.

## Key Point

CodeQL이 뭔지부터 이해할 필요가 있다. CodeQL은 소스 코드의 취약점을 자동으로 분석해주는 프로그램으로, 많은 쿼리들로 이루어져 있다. 이 쿼리들은 소스 코드를 분석하여 다양한 **데이터베이스**를 형성하고, 그 데이터베이스끼리 연산하여 최종 값을 도출해 내는 것이라 볼 수 있다.

여기서 **데이터베이스**에 주목할 필요가 있다. 해당 `example.qlx` 파일은 우리 코드를 분석하여 다양한 데이터베이스를 형성, 집합 연산하여 결국 `res` 값을 도출해 내는 것으로 생각할 수 있다. 이렇게 머리속에서 생각을 마친 뒤 해당 천 줄을 보게 되면 그렇게 어렵지 않다. 가령, 다음과 같은 데이터베이스는, `Left`, `Right` 값을 가지는 데이터베이스 하나를 형성한다고 볼 수 있겠다.

```
EVALUATE NONRECURSIVE RELATION:
  example::Shore#3b862151(unique string this) :- 
    {1} r1 = CONSTANT(unique string)["Left"]

    {1} r2 = CONSTANT(unique string)["Right"]

    {1} r3 = r1 UNION r2
    return r3
```

치명적인 문제점은 디버깅 수단이 알려지지 않았다는 것이다. 이에 대체재로 `[db]/log/execute-queries-[Date].log` 파일을 분석하여 왜 `wrong`이 나오는 것인지 검사할 수 있다.

이를 바탕으로 소스 코드를 짜서 서버에 제출하면 풀린다.

```bash
wane@wane:~/Hacking/ctf/lctf/sillake$ sha256sum main.c
1a1ca40c56398c9ea83d423f931e7833ead30b8369a556b9144aa21be92e10b4  main.c
```

`EPFL{r3v:3_ou_cauchem4r?}`