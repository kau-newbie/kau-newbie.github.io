---
layout: post
title:  "VerologHDL로 계산기 만들기(1)"
author: kau-newbie
categories: [ Verilog, Hardware, calculator ]
image: 
---

# Verilog로 계산기를 만들기(1)

## 개요

사실 처음만해도 무작정 계산기를 만들고 싶다고 생각했을 뿐이다. (만만해 보였던 건 사실)

계산기는 기본적으로 +(덧셈), -(뺄셈), *(곱셈), /(나눗셈)이 가능하고, 

operator들은 input으로 직접 switch 등을 이용해 넣어주며, (FPGA 보드를 가정하고 만들었다.)

어떤 모드를 선택할지도 input으로 받는다.

### I/O

계산기는 기본적으로 

### Schema(1)

우선, 처음 스펙을 정할 땐, 

단순히 덧셈 모듈, 뺄셈 모듈, 나눗셈 모듈, 곱셈 모듈 이렇게 4가지 모듈을 각자 따로 만들 생각이었다.

- Add controler

- Sub controler

- Mul controler

- Div controler

그러나, 이게 계산기라고 할 수 있을까? 자고로 계산기는 여러 산술을 할 수 있어야 한다고 생각한다.

해서, 덧셈, 뺄셈, 나눗셈, 곱셈을 통합하기로 했다.

- Top controler

    - Add controler

    - Sub controler

    - Mul controler

    - Div controler

대략적인 그림은 다음과 같았다.

![컨트롤러 부분 초안 - Controler part schema 1.0](../assets/images/cal_control.1.drawio.png)




