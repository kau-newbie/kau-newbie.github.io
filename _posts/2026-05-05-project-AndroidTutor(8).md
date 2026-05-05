---
layout: post
title:  "[project]AndroidTutor(8) 안드로이드 접근성 서비스에 관한 고찰-2"
author: kau-newbie
categories: [ android, project, accessibilityService, 접근성서비스, Runnable, Handler, Thread]
image: assets/images/prj-androidtutor-basicmodel.jpg
---

# 안드로이드 접근성 서비스에 관한 고찰-2

큰 문제가 생겼다. 우선, LLM에게 넘겨주는 ui 상태가 사용자가 보고 있는 현재 화면보다 조금 이전의 화면이다.
> 이전 포스팅에서도 썼지만, 우리 앱은 **'사용자의 화면 조작 --> ui 상태 변경 --> 화면 ui 상태(스냅샷)과 프롬프트를 LLM에게 보냄 --> LLM의 지시사항'의 흐름을 가진다.**
> - ui 상태가 변경되면 ui 정보를 트리순회하여 String으로 stateflow에 넘겨준다.
> - stateflow상태가 업데이트 되면, collect{...} 안의 코드가 실행되고, 이때 이 코드가 LLM에게 응답을 받아오는 코드이다.

예상되는 문제의 원인은 크게 두 가지이다.

1. 서비스 내 자체 필터링
- 서비스 내에서 TYPE_WINDOW_CONTENT_CHANGED의 이벤트 타입이면, 0.5초의 쿨타임을 가진다.
    - 즉, 0.5초 동안은 같은 타입의 이벤트를 받지 않는데, 실제 화면이 바뀔 때 (앱 실행이나, 앱 드로어가 열리거나, 등)에는 이런 content_changed가 여러번 발생한다.
- 그 중 첫 번째 타이밍의 ui 상태를 LLM에게 보내고 있는 것이다!

2. AccessibilityEvent.TYPE_TOUCH_INTERACTION_START
- 이것 또한 타이밍 문제인데, 사용자가 터치를 '시작하는 순간'부터 isTouched가 true가 된다. 터치를 '끝내는 순간'이 아닌 것이다.
- 물론 isTouched만으론 Ui 상태를(정보) stateFlow로 구현한 버퍼에 보내지 않는다. 
- 다만, 화면이 처음으로 바뀌는 순간 stateflow에 보낼 것이고, 곧바로 isTouched가 true이기 때문에 .collect{...}가 실행될 것이다.

이 중에서 2번은 사실상 큰 문제가 아니란 걸 나중에 알게됐다. 

왜냐하면 애초에 config.xml에서 `FLAG_SEND_MOTION_EVENTS` 플래그까지 설정했음에도 `TYPE_TOUCH_INTERACTION_X`는 감지하지 못했다.

**대부분의 사용자 인터렉션은 커스텀 뷰에서 뺏어간다.**는게 결론이다.
- 일부, 안드로이드 기본 뷰를 사용하는 경우에서만 `TYPE_VIEW_X` 타입의 이벤트들을 발생시킨다.
- 따라서 대부분의 이벤트들은 결국 isTouched를 ture로 바꾸면서(비록 사용자의 interaction은 감지하지 못했더라도), Ui 정보를 트리순회하여 stateFlow에게 보내야 한다.

결국 문제는 1번이었고, 이걸 해결하기 위해 **Debouncing**을 도입하려 한다.
> 일정 시간동안 이벤트들을 쌓아두고, 가장 최근 이벤트만을 처리하는 방식


## 접근성 이벤트 디바운싱 구현

물론, stateFlow.collect{...}앞에 .debouncing을 chainnig해서 달아둘 수도 있다.

하지만 여기서 중요한 건 '타이밍' 문제이다.

애초에 트리 구조를 순회하며 String을 만드느라 접근성 서비스(이하 서비스)는 느리다. 자체적으로 트리 구조를 recycle()한다지만, 최대한 만들지 않는 게 이득이다.

따라서 서비스에서 collect(외부 컴포넌트의 코드 안 어딘가)까지 넘어오기 전에, 애초에 서비스에서부터 필터링을 해주는 게 좋다는게 결론이다.

코린이인 나로서는 머리가 하얘진다. 젬나이 형님께 물어보자.

아무튼 서비스 내 디바운싱을 하기 위해 젬나이한테 물어보니 `Runnable`과 `Handler`를 알아야 한다고 했다.

### Runnable

[Runnable](https://developer.android.com/reference/java/lang/Runnable)

우선, 본문 번역이 다음과 같다. (LLM 시켰다.)

##### Runnable 인터페이스의 목적 

어떤 클래스가 Runnable을 구현하면, 그 클래스의 인스턴스는 스레드에 의해 실행될 수 있는 객체가 됩니다. 즉, 병렬 실행을 위해 공통된 규약을 제공하는 인터페이스입니다.

##### 필수 메서드 

Runnable을 구현하는 클래스는 반드시 인자가 없는 run() 메서드를 정의해야 합니다. 이 메서드 안에 스레드가 실행할 실제 작업 내용을 작성합니다.

##### Thread와의 관계

Thread 클래스 자체도 Runnable을 구현합니다.

하지만 Runnable을 직접 구현하면 Thread를 상속하지 않고도 실행 가능한 객체를 만들 수 있습니다.

예를 들어, Runnable을 구현한 객체를 Thread 생성자에 전달하면, 해당 객체의 run() 메서드가 스레드에서 실행됩니다.

##### 언제 Runnable을 쓰는가?

단순히 run() 메서드만 재정의하려는 경우에는 Runnable을 구현하는 것이 권장됩니다.

Thread를 상속하는 것은 그 클래스의 기본 동작을 바꾸거나 확장하려는 경우에만 적절합니다.

즉, 불필요하게 상속을 사용하는 대신 인터페이스 구현을 통해 더 깔끔하고 유연하게 스레드를 활용할 수 있습니다.

> 핵심 요약
> 
> - Runnable = 스레드에서 실행할 작업을 정의하는 인터페이스
>
> - 반드시 run() 메서드를 구현해야 함
>
> - Thread를 상속하지 않고도 스레드 실행 가능
>
> - 단순 실행 작업에는 Runnable 구현을, 스레드 자체 동작을 바꾸려면 Thread 상속을 사용

읽다보면 c의 `pthread`가 생각난다. 논리적으로 독립된 새로운 실행 흐름을 만들어준다. 단, 여기선 '행동'에 초점이 맞추어져있다.

Runnable.run() 메서드 안에 작성한 코드가 스레드의 실행 본체가 된다고 한다.
- 즉, c의 pthread_create()안에 넘겨주는 함수를 생각하면 이해가 빠를 것 같다.

#### Thread

그렇다면 [thread](https://developer.android.com/reference/java/lang/Thread?_gl=1*scm0ck*_up*MQ..*_ga*MjA0NTAxNDU2NS4xNzc3OTU3MjQ2*_ga_6HH9YJMN9M*czE3Nzc5NTcyNDUkbzEkZzAkdDE3Nzc5NTcyNDUkajYwJGwwJGg3MjQ4Njg2Mzg.)는 뭘까?

`A thread is a thread of execution in a program. The Java Virtual Machine allows an application to have multiple threads of execution running concurrently.`
> 말 그대로 분리된 새로운 실행 흐름이다.

본문을 읽다보면 c에서 보던 pthread처럼 똑같이 priority가 있고, thread가 thread를 만들고 (이때, 만든 thread의 priority로 initialized.)..., 별 다른 건 없다.

Java Virtual Machine(JVM)이 실행될 때, 처음에는 single non-daemon thread, 즉 main thread(안드로이드의 ui thread)만이 있다고 한다.

이 JVM은 다음과 같은 상황이 발생하기까지는 여러 thread들을 실행한다.
- The exit method of class Runtime has been called and the security manager has permitted the exit operation to take place.
- All threads that are not daemon threads have died, either by returning from the call to the run method or by throwing an exception that propagates beyond the run method.
> 모든 스레드가 죽거나, Runtime class에서 exit method를 실행했거나.

조금만 더 Thread를 살펴보자면, 여러 Thread를 만드는 방법이 있다.

```kotlin

class PrimeThread extends Thread {
    long minPrime;
    PrimeThread(long minPrime) {
        this.minPrime = minPrime;
    
        public void run() {
            // compute primes larger than minPrime
            . . .
        }
    }
}

PrimeThread p = new PrimeThread(143);
     p.start();

```

이때, 반드시 run()을 override해야 한다. 

`An instance of the subclass can then be allocated and started.`라는 걸 보니, instance하면 알아서 JVM에 의해 스케줄링 되는 것 같다.

#### 다시 Runnable()

Runnable로도 Thread를 만들 수 있댄다. 예시 코드를 살펴보자.

```kotlin

class PrimeRun implements Runnable {
    long minPrime;
    PrimeRun(long minPrime) {
        this.minPrime = minPrime;
    }

    public void run() {
        // compute primes larger than minPrime
        . . .
    }
}

PrimeRun p = new PrimeRun(143);
new Thread(p).start();
 
```

이때는 인자로 넘겨주는 형태이다. 지금 내 목적은 Handler를 통해 main Thread에게 이렇게 내가 정의한 run()코드를 넘겨줄 것이다.
> 위의 Runnable() 본문에서 보면, **단순히 run() 메서드만 재정의하려는 경우에는 Runnable을 구현하는 것이 권장됩니다.** 요게 핵심인 것 같은데, 
> 
> thread를 따로 만들지 않고, 이렇게 어떻게 분리된 실행 흐름이 *동작할지*만을 정하려면 Runnable을 쓰는 것 같다.
> - 새 thread를 만드는 게 아닌, main thread에게 다음에 할 일을(delay를 준다.) 쥐어주는 것이다. 그러면서 새로운 이벤트 발생시마다 기존 일은 취소를 시키는 식으로 구현하면 되겠다.

이쯤에서 깨달아요 젬나이 형님.... 멀티 스레딩이 중요한 게 아니고, main Thread에게 서비스 처리를 시키는 데, debouncing 동안 이전 작업을 계속 취소시키려고 그러신 거였군요.
- 어쩐지 나는 계속 공부하면서 멀티 스레딩? 코루틴이 있지 않나?하고 의심하고 있었다.
- 어차피 접근성 서비스는 Main Thread에서 동작한다. --> Main Thread만 관리하면 된다.
- 반면 코루틴의 Job 객체 관리는 복잡하다고 한다.
- LLM 말로는 Runnable객체 하나 재사용/ 교체 방식이 훨씬 가볍다고 하는데, 사실 이건 진지하게 고민해보아야 할 필요가 있을 것 같다.

사실 코루틴이 더 낫지 않나, 생각이 들긴 하는데, 그냥 공부도 할 겸, Handler로 Main Thread에게 Runanble()객체를 쥐어주는 방식으로 도전해보겠다.

코루틴 예시 코드는 대강 다음과 같이 이루어진다고 한다.

```kotlin

private var debounceJob: Job? = null
private val serviceScope = CoroutineScope(Dispatchers.Main) // 서비스용 스코프

override fun onAccessibilityEvent(event: AccessibilityEvent) {
    // 이전 작업 취소
    debounceJob?.cancel()

    // 새로운 작업 예약
    debounceJob = serviceScope.launch {
        delay(150L) // 0.15초 대기
        
        // 실행할 로직
        processEvent(event)
    }
}

```
어쨌거나 나는 계속 Handler + Runnable로 가겠다. 이제 다음은 Handler차례이다.

### Handler

`public class Handler`

