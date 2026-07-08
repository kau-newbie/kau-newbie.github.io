---
layout: post
title:  "[c] floating에 대한 정리"
author: kau-newbie
categories: [ c, memory, system ]
image: assets/images/forPost/c_logo.png
---

요약: c에서 floating point 고려사항

## float 

floating point(부동 소수점)은 IEEE에서 표준으로 비트 표기법을 정해놓았다.

$$Value = (-1)^s \times (1.fraction) \times 2^{(exponent - bias)}$$ 

혹은

$$Value = (-1)^s \times (M) \times 2^E $$

여기서 1.xxxx의 xxxx를 frac이 맡고 있다. 정확히는 frac에 1이 더해져있다고 가정한다.
> 예시로 많이 드는게 '과학적 표기법'이다. $1.x \times 10^n$ 형태를 띄고 있다.

일단 배울 때는 바로 위의 두 번째 표현으로 배웠으니 이 두 번째를 기준으로 삼겠다. 

M에서 이 (1 + frac)은 비트로 표현되고 있을테니, 2^E (E = exponent - bias)만큼 소숫점을 이동시킨다. 즉, 소숫점이 E칸만큼 이동한다. 요렇게 떠다닌다고 해서 'floating point'가 붙었다. 그리고 s는 sign bit (1/0)이다.

각 부분은 precision option에 따라 비트수가 정해져있다.

1. single precision: 32bits
> 1bit의 sign bit, 8bits의 exp, 23bits의 frac이다.

2. double precision: 64bits
> 1bit의 sign bit, 11bits의 exp, 52bits의 frac이다.

여기서 끝이 아니다. 표준에 따르면 `Normalized`와 `Denormalized`로 나뉜다.

1. `Normalized`
> 소숫점 바로 앞 수가 1이다. 1.xxx (1+frac이라 볼 수 있다.)
2. `Denormalized`
> 소숫점 바로 앞 수가 0이다. 0.xxx (0+frac이라 볼 수 있다.)
> - 조건은 `exp`가 `000...0`일 때이다.

**Normalized**

$exp \neq 000...0$ and $exp \neq 111...1$ 일 때,

E = exp - Bias이다. 
- `Bias`는 exp bits의 개수 k 일 때, $2^{k-1}-1$ 이다.

예로, 

single precision이면, exp bits는 8bits이므로, $2^{8-1}-1 = 2^{7}-1 = 127$ 이다.
double precision이면, exp bits는 11bits이므로, $2^{11-1}-1 = 2^{10}-1 = 1023$ 이다.

이렇게 하면, 최종적으로 $Value = (-1)^s \times (M) \times 2^E $ 에 따라 (M = 1 + frac 이므로,) $1.xxx \times 2^E$ 값이 나올 것이다.
- 이때, frac이 000...0 이면 $M = 1+ 0.0000$ (2진수이다.) 이므로 1.0이 나오겠다. 여기서 소숫점은 E만큼 이동할 것이다. (single precision이라 가정하면, E = exp - 127)
- 만약 frac이 111...1 이면 $M = 1 + 0.1111$ (역시 2진수이다.) 이므로 $2-\epsilon$ 이 frac이 되겠다.

이렇게 Normalized로는 frac이 00...0일 때, 최소 $-1 \times (2- \epsilon) \times 2^{(2^8-1-1)-127}$ 
- exp로 0x1111..110 이면 E = 255-1 - 127 = 127이 나온다.

그리고 최대 $1 \times (2-\epsilon) \times 2^{(127)}$ 까지 가능하다. 
- $\approx -3.4 \times 10^{38}$ ~ $(2 - 2^{-23}) \times 2^{127}$$\approx 3.4 \times 10^{38}$ 이 정도 범위를 가진다.

물론, 그 사이사이가 촘촘하지 않다! 비트 표현 한계 때문인데, 이 때문에 조금이라도 더 촘촘히 채워넣으려면 denormalized가 필요하다! 

**Denormalized**

숫자가 너무 작아 Normalized로 나타내지 못하는 경우에 쓴다.

$exp \eq 000...0$ 일 때 쓴다. 

우선, M = 0.xxxx이다.(2진수표현) 
- xxxx는 frac이니까, frac = 0000...0 이면, 0.00000을 나타내는 수이다. 단, 부호 비트가 있으니 +/- 0 표현이 가능하겠다.

단, E = exp- bias가 아닌, `E = 1-bias`이다.
- 딱 봐도 아주 작은 수를 표현하기 좋은 E이다.
- 생각해보면 exp가 모두 0으로 채워져있어서 exp는 , 더 나아가 E는 사실상 고정이다. `1-bias`로 E가 고정됐다.

**Special Values**

예약어처럼 정해진 값들이 있다. 

exp = 111...1 일 때, frac =000...0이면? `infinity`이다.

exp = 111...1 일 때, frac $\neq$ 0000..0이면? `NaN`이다. 알 수 없는 수라는 뜻이다.

결론적으로, 표현 범위를 생각하면, exp가 111...1 이 아닐 때,

Normalized 범위가 있고, 0 주위에 Denormalized의 범위가 밀집한 형태이다.