---
layout: post
title:  "[project]Project: AndroidTutor(5) coroutine(코루틴)와 flag 변수의 race condition"
author: kau-newbie
categories: [ android, os, coroutine, project]
image: assets/images/prj-androidtutor-basicmodel.jpg
---

# Android OS system

## 코루틴이 꼬이는 문제 및 flag 변수의 race condition

프로젝트 진행중에, 문제가 생겼다. 다음과 같은 상황이 발견됐다.

![이게무슨상황일까1](/assets/images/forPost/Androidsys/assoonas_getResponse1.png)
![이게무슨상황일까2](/assets/images/forPost/Androidsys/assoonas_getResponse2.png)
![이게무슨상황일까3](/assets/images/forPost/Androidsys/assoonas_getResponse3.png)

한 번 LLM의 응답을 받고난 거의 직후, 곧바로 중복되는 응답이 도착한다. 도착 시간이 각각 `2026-04-03 21:56:40.409` 와 `2026-04-03 21:56:45.584`이다. 가상머신 상인데도 체감상 2-3초밖에 되지 않았던 것 같다. 왜냐하면 애니메이션 로딩까지 조금 시간이 걸렸기 때문이다. 물론, 여기서 말하는 애니메이션은 현재 좌표 테스트용(LLM의 응답으로 받는 좌표를 빨간 네모영역으로 애니메이션을 띄운다.)과 실제 행동 유도용 손가락 애니메이션 중 전자이다. 후자는 실행되지도 않았다. 이전에도 이런 적이 있었는데, log상에서 그 이유를 알 수 있었다.
```
2026-04-03 21:56:40.410  1747-1747  KakaoTutor              com.mytutor.kakaotalktutorviews      D  [AnimationPlayer] playanimation 진입.
2026-04-03 21:56:40.411  1747-1747  KakaoTutor              com.mytutor.kakaotalktutorviews      D  [AnimationPlayer] playAnimation 종료시점.
2026-04-03 21:56:43.416  1747-1747  KakaoTutor              com.mytutor.kakaotalktutorviews      D  [GoalGuide] ===================================
                                                                                                    새로운 ui 스냅샷이 LogData.LogFlow에 업데이트된 시점.
                                                                                                    ===================================
2026-04-03 21:56:43.417  1747-1747  KakaoTutor              com.mytutor.kakaotalktutorviews      D  [GoalGuide] 
                                                                                                    현 시점 isTouched : true
                                                                                                     isMicUsed : true
2026-04-03 21:56:43.417  1747-1747  KakaoTutor              com.mytutor.kakaotalktutorviews      D  [GoalGuide] isMicused && isTouched 통과. 이제 false로 바꿈.
```
AnimationPlayer라는 모듈에서 애니메이션을 실행함과 동시에 취소했다. 그 이유는 명확하다. 바로 밑 줄에서 새로운 ui 스냅샷이 GoalGuide 모듈 안의 collect{}안에 넘겨졌기 때문이다. 이 collect{} 블록 안 코드들이 우리 앱의 주요 로직이다. 말그대로 계속해서 빙빙 돌면서 사용자의 목표를 달성할 때까지 LLM에게 질문 후 응답을 받아온다. 그 후 응답에 따라 애니메이션을 실행하는 코드로 넘어가게 되는데, 이 순간에 새 값이 들어와서 그렇다. 문제는 코루틴이 꼬였거나 (코루틴 실행시 이전 코루틴을 취소하는 코드가 있었거나), 무언가 코루틴을 취소하는 코드가 있는 것 같다. 어쨌거나 분명히 비정상적인 현상이다. 중복된 대답을 보낸다. (사실 애니메이션 중단은 비정상적인 일이 아니다. 당연히 최신 상태가 있거나, 사용자의 의도(목표 재설정)에 따라 기존 가이드 애니메이션은 취소돼야 한다.) 

그렇다면 collectLatest{}는 어떨까? 해당 메서드는 stateFlow의 값이 갱신될 때마다 블록 안 코드를 실행하는 건 같지만, 가장 최신 값만을 실행한다. 즉, 기존에 진행되던 블록 안 코드(이하 루틴이라고 하겠다.)를 새 값이 갱신되는 즉시 중단하고, 새로운 값에 대해 새 루틴을 실행한다.
```
...

    broadCastFlow.collectLatest{
        doIt()
        ...
    }

```
위와 같이 코드를 적어두면, broadCastFlow라는 stateFlow가 값이 갱신될 때마다, 이전 루틴(이전에 실행중이던 doIt())을 취소하고, 최신 갱신된 값에 대해 새 루틴(새 doIt())을 실행할 것이다. 하지만 문제가 있다. 이건 저번 학기에서 collectLatest --> collect로 바꾼 이유인데, 목표달성 검사 코드를 통과하면(목표를 달성했으면), 그 이후 한 번 더 코드를 보내는 와중에(쏟아지는 UI 상태 변경 이벤트 때문이다.) collectLatest 특성상 목표를 달성했다는 말풍선을 지워버리기 때문이다. 사용자에게 목표를 달성했다는 말풍선의 실행까지가 루틴의 완성(loop가 끝나는 시점)인데, 이걸 중단해 버린다. 그럼 collect{}로 바꿔버려도 여전히 문제는 남아있지 않느냐, 할 수도 있겠다. 이전 진행중이던 루틴을 취소하진 않겠지만, 또 여전히 목표를 달성했다고 두 번이나 중복을 보낼 여지가 있기 때문이다. 하지만 이건 루틴 도입부, 그러니까 colelct{}블록 초입부터 다음과 같은 플래그와, 지금 진행중인 코루틴이 이미 중단된 것인지 확인하는 일종의 중단점들을 군데군데 두었다.
```
(collect 블록 내부)

 if(LogData.isMicUsed && LogData.isTouched){...} //if문을 사용해서 막아두었다.

 ...

 currentCoroutineContext().ensureActive() // 취소해야할 때인지 확인. 중단점.
```
아무튼 이렇게 collect{}를 쓰게됐고, 문제점을 찾았다. 코루틴은 문제가 없다. 이전 루틴을(코루틴) 취소하는 코드는 (job객체의 .cancel()메서드를 썼다.) 기존에 진행하다 중단된 목표 불러오기를 했을 때(비정상 종료시를 위해 만든 기능이다.)와 버튼을 직접 사용자가 눌렀을 때 이다. instance by lazy로 인스턴스한 goalGuide는 (loop를 도는 루틴의 주체이다.) collect 특성상, 앞 루틴이 실행된 후에야 나중 루틴을 수행한다. 동시에 병렬로 수행하지 않는다. 그럼 어디가 문제였냐? 굳이 애니메이션에서 중단되는 문제는 .ensureActive때문이 아닌(즉, 다른 외부에서 의도적으로 해당 '코루틴'을 중지시킨 경우가 아니었다.), 바로 애니메이션 실행 코드 자체에 있었다.
```
//40초 동안 사용자의 터치가 없다면, 혹은 사용자가 터치한다면
                withTimeoutOrNull(40000L) {
                    LogData.logFlow.first { LogData.isTouched }
                }
                //넘어가서 실행
                animationPlayer.removeGuide(windowManager)
```
어쩌면 당연한게, 사용자가 가이드 애니메이션을 따라 터치를 했을 경우(올바른 좌표인지 아닌지는 확인하지 않는다. 해당 좌표빼고 모두 블락한다면, (애초에 그럴 수 있나?) 뭔가 기분이 찜찜해서 안 넣었다. 가이드라기보다는 무언가 강제하는 체벌같다.) 애니메이션은 즉시 종료된다. 그럼 사용자가 터치를 했었느냐? 나는 분명 터치를 하지 않았고, 로그만 봐도 정말 순식간에 터치가 이루어졌다. 그렇다. 코드 문제이다. 접근성 서비스에서 사용자 터치를 감지했기 때문인데, 이건 내 코드에 오류가 있어서 그렇다. 앞에서 말했듯 특정 view들은 사용자 터치를 반응하지 않고, 따라서 나는 TYPE_WINDOWS_CHANGED(안드로이드 시스템에서 window 단위의 변화가 있을 때) 플래그가 있을 때면 isTouched 라는 터치 플래그를 true로 바꾸게끔 설계했다. 범인을 찾았다.

내 앱에서, 접근성 이벤트는 Ui 상태(이하 '상태')의 갱신인데, 이때마다 collect{} 루틴을 돌게할 것이다. 그렇다고 이벤트가 쏟아지는 (나름 고심해서 선택한다고는 했지만, 여전히 많다. 아직 수정해야할 것이다.) 상황에서 이벤트 감지 범위를 줄이자니 팝업창이라던가, 뭔가 특별한(접근성 서비스 대부분을 발생시키지 않는 view- 예를 들면 터치를 절대 감지하지 않는다.) view들에 대해 감지를 못하는 경우가 있어 어느정도 감내해야만 했다. 물론 이대론 그냥 둘 수 없다. 지금 원하는 수정 방향은 기존에 진행 중이던 대답과 중복이면 아예 새로 시작될 루틴을 취소해 버리는 경우이다. 이 방법은 루틴 중단 시점에 따라 나눌 수 있을 것 같다.
1. 상태 갱신 시점에서부터, 즉 접근성 서비스의 이벤트를 보고, 대답이 중복일 것 같은 경우인지 판단한다. 
2. 앞선 루틴이 있으면 상태 갱신 후, collect{}에 앞선 루틴이 모두 실행되기를 기다리는 중에 취소를 판단한다.
3. 대답을 확인하고 코루틴을 중지시키거나 화면에 반영하지 않는다.

3번은 너무 느리다. 어느세월에 다시 LLM에게 쿼리를 보내고, 추론시간을 기다려 완성된 응답을 받아오지? 이미 accessibility_service_config.xml파일에서 android:notificationTimeout="300" 해당 flag 값을 0.3초로 설정해뒀다. 물론, 이것보다는 collect에서 .debounce(3000) 메소드로 3초간 stateFlow에서 갱신되는 상태 값들의 반영을 지연시키기 때문이다.