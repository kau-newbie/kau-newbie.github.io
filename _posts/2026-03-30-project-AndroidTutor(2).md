---
layout: post
title:  "[project]Project: AndroidTutor(2) Hilt에 대해서"
author: kau-newbie
categories: [ project, android, app, ai ]
image: assets/images/prj-androidtutor-basicmodel.jpg
---

# AndroidTutor(가제) 프로젝트(2)

여러모로 기워붙이며 기능을 추가한지 어언 몇 달째...,

현재 코드의 상태는 다음과 같았다.

![난잡한 코드의 상태](../assets/images/forPost/AndroidTutor(2)/massy-modules.png)

둘다 reterofit을 써서, 같은 OpenAi 서버에게 보내는데 각기 다른 모듈을 만들어 쓰고 있었다. endpoint와 모델, 모델변수 등만 다른데, 뭔가 중복을 줄일 방법이 없을까? 어디서 주워들은 건 있어가지고, annotation(@)을 적어두면 코드가 알아서 이때는 A에게 보냈다가, 이때는 B에게 보내는 무언가를 찾아봤다. 젬나이씨의 말로는 Hilt를 쓰면 된다고 했다.

## HILT란?

참고: 안드로이드 개발자페이지 (https://developer.android.com/training/dependency-injection/hilt-android?hl=ko)

Hilt란 위 페이지에 따르면,

```
Hilt는 프로젝트에서 종속 항목 수동 삽입을 실행하는 상용구를 줄이는 Android용 종속 항목 삽입 라이브러리입니다. 종속 항목 수동 삽입을 실행하려면 모든 클래스와 종속 항목을 수동으로 구성하고 컨테이너를 사용하여 종속 항목을 재사용 및 관리해야 합니다.

Hilt는 프로젝트의 모든 Android 클래스에 컨테이너를 제공하고 수명 주기를 자동으로 관리함으로써 애플리케이션에서 DI를 사용하는 표준 방법을 제공합니다. Hilt는 Dagger가 제공하는 컴파일 시간 정확성, 런타임 성능, 확장성 및 Android 스튜디오 지원의 이점을 누리기 위해 인기 있는 DI 라이브러리인 Dagger를 기반으로 빌드되었습니다. 자세한 내용은 Hilt 및 Dagger를 참조하세요.
```
라고 한다. 여기서 다시, '종속 항목 삽입'이란 뭘까?

영어로는 dependency injection 인데, 주로 사람들이 쓸 때 '의존성 주입'이라 했었다. 해서 앞으로는 의존성 주입으로 말하겠다.

