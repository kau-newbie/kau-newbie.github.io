---
layout: post
title:  "[project]Project: AndroidTutor(2) Hilt에 대해서"
author: kau-newbie
categories: [ project, android, app, DI, Hilt ]
image: assets/images/prj-androidtutor-basicmodel.jpg
---

# AndroidTutor(가제) 프로젝트(2)

여러모로 기워붙이며 기능을 추가한지 어언 몇 달째...,

현재 코드의 상태는 다음과 같았다.

![난잡한 코드의 상태](../assets/images/forPost/AndroidTutor(2)/massy-modules.png)

둘다 reterofit을 써서, 같은 OpenAi 서버에게 보내는데 각기 다른 모듈을 만들어 쓰고 있었다. endpoint와 모델, 모델변수 등만 다른데, 뭔가 중복을 줄일 방법이 없을까? 어디서 주워들은 건 있어가지고, annotation(@)을 적어두면 코드가 알아서 이때는 A에게 보냈다가, 이때는 B에게 보내는 무언가를 찾아봤다. 젬나이씨의 말로는 Hilt를 쓰면 된다고 했다.

## HILT란?

참고: 안드로이드 개발자 페이지 (https://developer.android.com/training/dependency-injection/hilt-android?hl=ko)

**Hilt란** 위 페이지에 따르면,
```
Hilt는 프로젝트에서 종속 항목 수동 삽입을 실행하는 상용구를 줄이는 Android용 종속 항목 삽입 라이브러리입니다. 종속 항목 수동 삽입을 실행하려면 모든 클래스와 종속 항목을 수동으로 구성하고 컨테이너를 사용하여 종속 항목을 재사용 및 관리해야 합니다.

Hilt는 프로젝트의 모든 Android 클래스에 컨테이너를 제공하고 수명 주기를 자동으로 관리함으로써 애플리케이션에서 DI를 사용하는 표준 방법을 제공합니다. Hilt는 Dagger가 제공하는 컴파일 시간 정확성, 런타임 성능, 확장성 및 Android 스튜디오 지원의 이점을 누리기 위해 인기 있는 DI 라이브러리인 Dagger를 기반으로 빌드되었습니다. 자세한 내용은 Hilt 및 Dagger를 참조하세요.
```
라고 한다. 여기서 다시, '종속 항목 삽입'이란 뭘까? (자동번역이라 그런듯 하다.)

## Dependency Inejction 이란?

영어로는 **dependency injection** 인데, 주로 사람들이 쓸 때 '의존성 주입'이라 했었다. 해서 앞으로는 의존성(요소) 주입으로 말하겠다.
역시나, 정의를 찾기 위해 안드로이드 개발자 페이지를 참고했다. (https://developer.android.com/training/dependency-injection?hl=ko)
```
클래스에는 흔히 다른 클래스 참조가 필요합니다. 예를 들어 Car 클래스는 Engine 클래스 참조가 필요할 수 있습니다. 이처럼 필요한 클래스를 의존성 요소라고 하며, 이 예에서 Car 클래스가 실행되기 위해서는 Engine 클래스의 인스턴스가 있어야 합니다.
```
그렇다. 여기서 어느정도 dependency injection에 관한 힌트가 나왔는데, 클래스에 필요한 클래스를 **'종속 항목'**이라고 한다는 것이다.
조금만 더 살펴보자. 이런 의존성 요소는 클래스가 객체로 인스턴스 되기 위해, 해당 요소를 객체로서 가져와야 한다. 혹은, 적어도 static 메모리에 올라가기 위해, 해당 의존성 요소가 꼭 필요하다. 안드로이드 개발자 페이지에서는, 다음과 같은 세 가지로 객체를 얻을 수 있다고 설명한다.
1. 클래스가 필요한 의존성 요소를 구성합니다. 위의 예에서 Car는 자체 Engine 인스턴스를 생성하여 초기화합니다.
2. 다른 곳에서 객체를 가져옵니다. Context getter 및 getSystemService()와 같은 일부 Android API는 이러한 방식으로 작동합니다.
3. 객체를 매개변수로 제공받습니다. 앱은 클래스가 구성될 때 이러한 의존성 요소를 제공하거나 각 의존성 요소가
 필요한 함수에 전달할 수 있습니다. 

대표적으로 '합성'이 생각난다. 이를테면 Computer{} class안의 Mouse{}객체, Keyboard{}객체 등이다. 그렇다고 has-a 관계이면(합성) 해당 객체들은 모두 의존성 요소인가? 이건 제미나이가 알려줬다.
```
요약하자면
의존성 주입에서 말하는 **'의존성(Dependency)'**은 "합성" 관계에 있는 객체만을 의미하는 것이 아니라, **"이 클래스가 제대로 작동하기 위해 외부에서 공급받아야 하는 모든 것"**을 통칭합니다.
```
> 그렇다. 정리하자면, **외부에서 전달받아 참조해야할 내부 구성자**는 모두 **'의존성(요소)'**인 것이다.

아무래도 객체지향이다 보니, 메모리 공간상의 개념이 dependency를 이해하는데 도움이 될 것 같아 메모리에 연관지어 물어봤다. 제미나이의 답변은 다음과 같았다.
```
컴퓨터 구조에서 객체는 Heap(힙) 메모리에 할당됩니다.

1. 메모리 배치 모습
val myComputer = Computer(Mouse(), Keyboard()) 코드가 실행될 때의 모습은 대략 이렇습니다.

Heap 영역 A: Mouse 객체 데이터 (실제 값들)

Heap 영역 B: Keyboard 객체 데이터 (실제 값들)

Heap 영역 C: 
    
    Computer 객체 데이터

    mouseReference: 0x123 (영역 A의 주소)

    keyboardReference: 0x456 (영역 B의 주소)

핵심: Computer 클래스의 크기는 내부 멤버 변수의 개수(참조 변수의 크기)에 따라 결정되며, 그 변수들이 가리키는 대상(Mouse, Keyboard)은 메모리의 전혀 다른 번지수에 떨어져서 존재합니다.
```
훨씬 직관적이고 잘 이해가 갔다. *메모리 상에서 서로 다른 별개의 공간에 위치한 어떤 한 객체와 그 객체의 내부 구성원들이 있다. 이때 이 내부 구성원들을 의존성(요소)-dependency라 부른다.*

어느 블로그를 가나 설명하는 Car 예시를 한 번 살펴보겠다. (다 사이트에 나와있는 예제이다.) 아래는 dependency injection이 아니라(이하 DI) dependency를 직접 내부 생성으로 얻는 방식이다.
```java
class Car {

    private val engine = Engine()

    fun start() {
        engine.start()
    }
}

fun main(args: Array) {
    val car = Car()
    car.start()
}
```
이번에는 DI 방식을 보겠다.
```java
class Car(private val engine: Engine) {
    fun start() {
        engine.start()
    }
}

fun main(args: Array) {
    val engine = Engine()
    val car = Car(engine)
    car.start()
}
```
확실히 차이가 보인다! 외부에서 의존성(요소)를 넣어주고 있다.

## HILT란?(2)

자, 다시 넘어와서, 