---
title: Using l'Hôpital's rule
published: 2024-10-12
description: "Solve limit problems with l'Hôpital's rule"
image: '/l/l.jpg'
tags: ['Math']
category: 'Calculus'
draft: false 
---

# Using l'Hôpital's rule

간단한 수학 얘기이다. 로피탈의 정리를 활용하여 극한 문제를 풀자.

로피탈의 정리는 다음과 같다.

> 정의 (간소화): 미분가능한 함수 $f, g: \mathbb{R} \to \mathbb{R}$ 에서 $\lim_{x \to  a}f(x) = 0$ 혹은 $\lim_{x \to  a}|f(x)| = \infty$ 이고, $\lim_{x \to  a}g(x) = 0$ 혹은 $\lim_{x \to  a}|g(x)| = \infty$ 일 경우에 다음이 성립한다.
> $$\lim_{x \to  a}\frac{f(x)}{g(x)} = \lim_{x \to  a}\frac{f'(x)}{g'(x)}$$

유명 강사 '현' 씨께서는 쓰지 말라 당부 하셨지만 안 쓸 이유가 없을 뿐더러 안 쓰면 손해이다.

## l'Hopital's Rule II

다음이 성립한다.

> l'Hopital's Rule II:
> $$\lim_{x \to  a}\frac{f(x)}{g(x)} = \lim_{x \to  a}\frac{f''(x)}{g''(x)}$$

증명은 삼단논법이다. 이계도함수 뿐만 아니라 로피탈의 정리를 만족하는 경우 삼계, 사계도함수까지도 성립한다.

따라서 다음과 같은 알고리즘으로 **모든** 극한 문제를 풀 수 있다.

> 미분가능한 함수 $f(x)$, $g(x)$에 대하여 $\lim_{x \to a}\frac{f(x)}{g(x)}$를 구하기 위해 $f(x)$의 $n$계도함수 $f_n(x)$에 대해 $\lim_{x \to a}f_n(x) \neq 0$거나, $g(x)$의 $n$계도함수 $g_n(x)$에 대해 $\lim_{x \to a}g_n(x) \neq 0$ 인 $n$을 계산하고 $\lim_{x \to a}\frac{f_n(x)}{g_n(x)}$ 를 구한다.

거의 모든 경우에서 $n=1$이다.

## Application

예제 몇 가지를 풀어보자.

$$\lim_{x \to 1}\frac{x^4 - 3x^3 - 13x^2 + 51x - 36}{14x^3 + 25x^2 - 12x - 27}$$

를 구해보자.

$f(x) = x^4 - 3x^3 - 13x^2 + 51x - 36$ 에 대해 $f'(x) = 4x^3 - 9x^2 - 26x + 51$이고, $g(x) = 14x^3 + 25x^2 - 12x - 27$ 에 대해 $g'(x) = 42x^2 + 50x - 12$ 이다.

$$\lim_{x \to 1}\frac{4x^3 - 9x^2 - 26x + 51}{42x^2 + 50x - 12} = \frac{1}{4}$$

답은 $\frac{1}{4}$ 이다.

다른 예를 들어보자.

$$\lim_{x \to 0+}\frac{73^x+\log_{37}x}{37^x+\log_{73}x}$$

$f(x) = 73^x+\log_{37}x$ 에 대해 $\lim_{x \to 0+}f(x)=-\infty$ 이고 $g(x) = 37^x+\log_{73}x$ 에 대해 $\lim_{x \to 0+}g(x)=-\infty$ 이니 로피탈의 정리를 쓸 수 있다.

$$f'(x) = \frac{73^xx\ln37\ln73+1}{x\ln37}$$

$$g'(x) = \frac{37^xx\ln37\ln73+1}{x\ln73}$$

$$\frac{f'(x)}{g'(x)} = \frac{(37^xx\ln37\ln73+1)x\ln73}{(37^xx\ln37\ln73+1)x\ln37}$$

$$\lim_{x \to 0+}\frac{f'(x)}{g'(x)} = \frac{\ln73}{\ln37} = \log_{37}{73} = \lim_{x \to 0+}\frac{f(x)}{g(x)}$$

이와 같이 문제를 아주 빠르게 풀 수 있다.

## Caution

무지성 로피탈은 문제 풀이 속도가 느려질 수 있다. 특히 알고리즘에서 만족하는 $n$이 3 이상일 때이다. 예를 들어보자.

$$ \lim_{x \to 0}\frac{\cos^2x - 2\cos x + 1}{2x^4} $$

이때 알고리즘에 따르면 만족하는 $n = 4$이다. 이때 로피탈의 정리를 사용하면 문제 풀이 시간이 많이 걸린다.

$$\lim_{x \to 0}\frac{\cos^2x - 2\cos x + 1}{2x^4} = \lim_{x \to 0}\frac{1}{2} \times \frac{(\cos x - 1)^2}{(x^2)^2}$$

$$\lim_{x \to 0}\frac{1}{2} \times \frac{(\cos x - 1)^2}{(x^2)^2} = \lim_{x \to 0}\frac{1}{2} \times (\frac{\cos x - 1}{x^2})^2$$

$$\lim_{x \to 0}\frac{1}{2} \times (\frac{\cos x - 1}{x^2})^2 = \lim_{x \to 0}\frac{1}{2} \times (\frac{1}{2})^2 = \frac{1}{8}$$

로 구하는 것이 더 쉽고 빠르다. 무지성 로피탈은 수학 3등급이나 하는 것이니 피하도록 하자.