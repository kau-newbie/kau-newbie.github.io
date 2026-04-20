---
layout: post
title:  "[project]AndroidTutor(2) Hilt로 migration하자-1"
author: kau-newbie
categories: [ project, android, app, DI, Hilt ]
image: assets/images/prj-androidtutor-basicmodel.jpg
---

# DI란?

여러모로 기워붙이며 기능을 추가한지 어언 몇 달째...,

현재 코드(일부)의 상태는 다음과 같았다.

![난잡한 코드의 상태](../assets/images/forPost/AndroidTutor(2)/massy-modules.png)

OpenAiApi나 ChatGpt 모듈이나 둘다 reterofit을 써서, 같은 OpenAi 서버에게 보내는데, 각기 다른 모듈로 존재하고 있었다. endpoint와 모델, 모델변수 등만 다른데, 뭔가 중복을 줄일 방법이 없을까? 어디서 주워들은 건 있어가지고, annotation(@)을 적어두면 코드가 알아서 이때는 A에게 보냈다가, 이때는 B에게 보내는 무언가를 찾아봤다. 젬나이씨의 말로는 Hilt를 쓰면 된다고 했다.

## HILT란?

참고: [안드로이드 개발자 페이지](https://developer.android.com/training/dependency-injection/hilt-android?hl=ko)

**Hilt란** 위 페이지에 따르면,
```
Hilt는 프로젝트에서 종속 항목 수동 삽입을 실행하는 상용구를 줄이는 Android용 종속 항목 삽입 라이브러리입니다. 종속 항목 수동 삽입을 실행하려면 모든 클래스와 종속 항목을 수동으로 구성하고 컨테이너를 사용하여 종속 항목을 재사용 및 관리해야 합니다.

Hilt는 프로젝트의 모든 Android 클래스에 컨테이너를 제공하고 수명 주기를 자동으로 관리함으로써 애플리케이션에서 DI를 사용하는 표준 방법을 제공합니다. Hilt는 Dagger가 제공하는 컴파일 시간 정확성, 런타임 성능, 확장성 및 Android 스튜디오 지원의 이점을 누리기 위해 인기 있는 DI 라이브러리인 Dagger를 기반으로 빌드되었습니다. 자세한 내용은 Hilt 및 Dagger를 참조하세요.
```
라고 한다. 여기서 다시, '종속 항목 삽입'이란 뭘까? (자동번역이라 그런듯 하다.)

## Dependency Inejction 이란?

영어로는 **dependency injection** 인데, 주로 사람들이 쓸 때 '의존성 주입'이라 했었다. 해서 앞으로는 의존성(요소) 주입으로 말하겠다.
역시나, 정의를 찾기 위해 안드로이드 개발자 페이지를 참고했다. [관련 개발자 페이지](https://developer.android.com/training/dependency-injection?hl=ko)
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

계속해서 공식문서를 읽어보자.
```
There are two major ways to do dependency injection in Android:

Constructor Injection. This is the way described above. You pass the dependencies of a class to its constructor.

Field Injection (or Setter Injection). Certain Android framework classes such as activities and fragments are instantiated by the system, so constructor injection is not possible. With field injection, dependencies are instantiated after the class is created. The code would look like this:
```
아무래도 DI는 클래스를 인스턴스할 때 사용하는 방법인가보다. (필요한 내부 구성요소가 없어서 에러가 나면 안되니까!)
보통 두 가지 DI의 방법이 있다고 한다.
1. 바로 위 예시처럼 생성자에 외부에서 넘겨받은 객체 인자를 넘겨줄 때
2. 1번이 안되는 상황: Activity나 Fragments같은 안드로이드 시스템(OS)자체가 객체로 만드는(instance화하는) 경우, 즉 필드 주입(field injection)을 해야할 때


이때, 필드 주입이란, 내부의 변수(=필드)에 나중에 채워넣는 방식이다. 어떻게 나중에 채워넣는단 말인가요? 한다면, 나도 마침 프로젝트를 진행하면서 꽤 자주 본 예약어가 있다. 바로 `lateinit`이다. 
> 처음 실행 때 블록 안 코드를 실행하고, 그 다음부터는 재사용하는 `lazy`와는 다르다.

예시를 보자.
```java
class LoginActivity : AppCompatActivity() {
    
    // Activity가 시스템에 의해 생성된 후, Hilt가 이 필드에 ViewModel 주소를 꽂아줌
    @Inject
    lateinit var viewModel: LoginViewModel 

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // 이제 viewModel 사용 가능
    }
}
```
LoginActivity라는 클래스를 인스턴스하기 위해선, 내부 구성요소인 viewModel이 필요하다. 문제는 viewModel은 Activity가 만들어지며 system에서 따로 만들어주는 객체이다. 이처럼 Service, Activity, Fragments 등 시스템이 객체를 만들어주는 클래스들은 lateinit var(반드시 가변 변수 var로 선언해야 한다고 한다.)을 이용해 내부 변수(field)로 선언을 해놓고, 후에 Hilt같은 DI 툴들이 넣어주도록 해야한다는 것이다. 제미나이는 아래처럼 설명했다.
```
메모리 관점

1. Activity 객체가 힙(Heap)에 먼저 덩그러니 올라갑니다.

2. 이후 DI 툴이 Activity 안에 있는 loginViewModel이라는 **리모컨(주조값 저장소)**에 실제 LoginViewModel 객체의 주소값을 "나중에" 딱 꽂아주는 것입니다.
```
여기까지 dependencies를 주입하는 것까지 알아보았다. 문제는, 이게 너무 많아지면? dependencies들이 아주 방대하고, 또 복잡하게 얽혀있으면? 안드로이드 페이지에서는 그래프 같은 것들로 관리하는 것을 권장한다. 그리고 추가로 - 여기서 중요하다 - 이걸 모두 해결(관리)해주는 **library들을 소개한다.**
> 이 라이브러리는 
> 1. 정적 타임 (컴파일 시)
> 2. 런 타임 (실행 중)
> 에서 모두 dependency injection을 지원한다. 

**Dagger**

Dagger 가 여기서 가장 대표적인 라이브러리 중 하나라고 한다. 
```Dagger facilitates using DI in your app by creating and managing the graph of dependencies for you. It provides fully static and compile-time dependencies addressing many of the development and performance issues of reflection-based solutions such as Guice.```
하지만 나는 Dagger를 더 쉽고 편하게 쓸 수 있게 해준 Hilt로 바로 넘어갈 것이다.

**Service Locator**

Service Locator는 DI tool(library)가 아닌, DI의 대안이다. 자세히 알지 못해(원문도 언급만 한다.) 가볍게 짚고만 넘어갈 것이다.
원문에서 보여준 예시코드는 다음과 같다.
```java
object ServiceLocator {
    fun getEngine(): Engine = Engine()
}

class Car {
    private val engine = ServiceLocator.getEngine()

    fun start() {
        engine.start()
    }
}

fun main(args: Array) {
    val car = Car()
    car.start()
}
```
다만, DI에 비해 단점이 크게 두 가지 정도 있었는데,
1. 테스트가 어렵다.
- 보통 싱글톤 객체로 locator를 만들기 때문에, 각 객체별로 분리가 되는(예를 들어, Car예시에서는 매 인스턴스 시점마다 서로 다른 가짜 클래스(테스트용)를 만들어 넣어줄 수 있다.) DI와 달리, 따로 코드를 추가하거나 바꾸지 않았다면 계속 같은 가짜 클래스를 넣어주게 된다.
2. 의존성(요소)을 파악하기 어렵게 한다. 
- DI면 클래스의 인자(arguments)로 바로 파악이 가능한데, 일일히 코드 안을 들여다 봐야(첫째 줄부터 바로 볼 수 있지 않다.) 어떤 의존성(요소)를 필요로 하는지 알 수 있게 된다.

## 수동 DI v.s. container 방식?

Hilt로 넘어가기 전에 뭐 하나를 찾아봐야만 했다. Hilt 안내 문서에서는 가장 먼저 앱에서 Application 클래스가 필요했는데, 이게 왜 필요한지 찾아보니, 수동 DI에 관한 문서를 읽어봐야했다. [수동 DI 문서](https://developer.android.com/training/dependency-injection/manual?hl=ko&_gl=1*1jcx920*_up*MQ..*_ga*MTE0NDQ2Mjk1LjE3NzQ4NTI3ODI.*_ga_6HH9YJMN9M*czE3NzQ4NTI3ODEkbzEkZzAkdDE3NzQ4NTI3ODEkajYwJGwwJGg1ODkwNjY4ODA.)에 따르면,
```
클래스 간 종속성은 그래프로 표시할 수 있고 그래프에서 각 클래스는 종속된 클래스에 연결됩니다. 모든 클래스와 서로의 종속성을 표시하면 애플리케이션 그래프가 구성됩니다. 

종속 항목 삽입을 사용하면 클래스를 쉽게 연결할 수 있고 테스트를 위해 구현을 교체할 수 있습니다. 예를 들어 저장소에 종속된 ViewModel을 테스트할 때 가짜 또는 모의 구현과 함께 Repository의 다른 구현을 전달하여 다른 사례를 테스트할 수 있습니다. <--이건 이미 알고 있는 사실이다.
```
라고 한다. 해서, 내가 팀원들과 개발중인 앱도, 이번에 hilt로의 migration(이라도 해도 되려나요)을 하는 목적과, 그 목표를 명확히 할겸, 그래프를 그려보겠다. 아래와 같다.

![Dependency Graph over the App]()

**Hilt**

드디어 힐트이다. Dagger를 기반으로 훨씬 쉽게 사용할 수 있게 만들었다. 공식 안드로이드 페이지에서도 권장한다고 나와있다. 

자, 다시 넘어와서, 

[의존성주입-안드로이드 공식문서](https://developer.android.com/training/dependency-injection/hilt-android?hl=ko&_gl=1*49kihh*_up*MQ..*_ga*MTU4ODYxOTE0LjE3NzQ4NjUwNDc.*_ga_6HH9YJMN9M*czE3NzQ4NjUwNDYkbzEkZzAkdDE3NzQ4NjUwNDYkajYwJGwwJGg0NDQyMzYzMjU)
의 내용을 따라 계속해서 진행해보자. 물론, 나에겐 쉽지 않은 여정이었다.


...To Be Continue....