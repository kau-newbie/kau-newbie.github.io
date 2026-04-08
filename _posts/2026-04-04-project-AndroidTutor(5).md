---
layout: post
title:  "[project]Project: AndroidTutor(5) coroutine(코루틴)와 flag 변수의 race condition"
author: kau-newbie
categories: [ android, os, coroutine, project]
image: assets/images/prj-androidtutor-basicmodel.jpg
---

# Android OS system

## 코루틴이 꼬이는 문제 및 flag 변수의 race condition

### collect{}와 collectLatest{}

프로젝트 진행중에, 문제가 생겼다. 다음과 같은 상황이 발견됐다.

![이게무슨상황일까1](/assets/images/forPost/Androidsys/assoonas_getResponse1.png)
![이게무슨상황일까2](/assets/images/forPost/Androidsys/assoonas_getResponse2.png)
![이게무슨상황일까3](/assets/images/forPost/Androidsys/assoonas_getResponse3.png)

한 번 LLM의 응답을 받고난 거의 직후, 곧바로 중복되는 응답이 도착한다. 도착 시간이 각각 `2026-04-03 21:56:40.409` 와 `2026-04-03 21:56:45.584`이다. 가상머신 상인데도 체감상 2-3초밖에 되지 않았던 것 같다. 왜냐하면 애니메이션 로딩까지 조금 시간이 걸렸기 때문이다. 

물론, 여기서 말하는 애니메이션은 현재 좌표 테스트용(LLM의 응답으로 받는 좌표를 빨간 네모영역으로 애니메이션을 띄운다.)과 실제 행동 유도용 손가락 애니메이션 중 전자이다. 후자는 실행되지도 않았다. 이전에도 이런 적이 있었는데, log상에서 그 이유를 알 수 있었다.
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
AnimationPlayer라는 모듈에서 애니메이션을 실행함과 동시에 취소했다. 그 이유는 명확하다. 바로 밑 줄에서 새로운 ui 스냅샷이 GoalGuide 모듈 안의 collect{}안에 넘겨졌기 때문이다. 
> 이 collect{} 블록 안 코드들이 우리 앱의 주요 로직이다. 말그대로 계속해서 빙빙 돌면서 사용자의 목표를 달성할 때까지 LLM에게 질문 후 응답을 받아온다. 
> 
> 그 후 응답에 따라 애니메이션을 실행하는 코드로 넘어가게 되는데, 이 순간에 새 값이 들어와서 그렇다. 

문제는 코루틴이 꼬였거나 (코루틴 실행시 이전 코루틴을 취소하는 코드가 있었거나), 무언가 코루틴을 취소하는 코드가 있는 것 같다. 어쨌거나 분명히 비정상적인 현상이다. 중복된 대답을 보낸다. 
> (사실 애니메이션 중단은 비정상적인 일이 아니다. 당연히 최신 상태가 있거나, 사용자의 의도(목표 재설정)에 따라 기존 가이드 애니메이션은 취소돼야 한다.) 

그렇다면 collectLatest{}는 어떨까? 해당 메서드는 stateFlow의 값이 갱신될 때마다 블록 안 코드를 실행하는 건 같지만, 가장 최신 값만을 실행한다. 즉, 기존에 진행되던 블록 안 코드(이하 루틴이라고 하겠다.)를 새 값이 갱신되는 즉시 중단하고, 새로운 값에 대해 새 루틴을 실행한다.
```
...

    broadCastFlow.collectLatest{
        doIt()
        ...
    }

```
위와 같이 코드를 적어두면, broadCastFlow라는 stateFlow가 값이 갱신될 때마다, 이전 루틴(이전에 실행중이던 doIt())을 취소하고, 최신 갱신된 값에 대해 새 루틴(새 doIt())을 실행할 것이다. 

하지만 문제가 있다. 이건 저번 학기에서 collectLatest --> collect로 바꾼 이유인데, 목표달성 검사 코드를 통과하면(목표를 달성했으면), 그 이후 한 번 더 코드를 보내는 와중에(쏟아지는 UI 상태 변경 이벤트 때문이다.) collectLatest 특성상 목표를 달성했다는 말풍선을 지워버리기 때문이다. 사용자에게 목표를 달성했다는 말풍선의 실행까지가 루틴의 완성(loop가 끝나는 시점)인데, 이걸 중단해 버린다. 

그럼 collect{}로 바꿔버려도 여전히 문제는 남아있지 않느냐, 할 수도 있겠다. 이전 진행중이던 루틴을 취소하진 않겠지만, 또 여전히 목표를 달성했다고 두 번이나 중복을 보낼 여지가 있기 때문이다. 

하지만 이건 루틴 도입부, 그러니까 colelct{}블록 초입부터 다음과 같은 플래그와, 지금 진행중인 코루틴이 이미 중단된 것인지 확인하는 일종의 중단점들을 군데군데 두었다.
```
(collect 블록 내부)

 if(LogData.isMicUsed && LogData.isTouched){...} //if문을 사용해서 막아두었다.

 ...

 currentCoroutineContext().ensureActive() // 취소해야할 때인지 확인. 중단점.
```
아무튼 이렇게 collect{}를 쓰게됐고, 문제점을 찾았다. 

코루틴은 문제가 없다. 이전 루틴을(코루틴) 취소하는 코드는 (job객체의 .cancel()메서드를 썼다.) 기존에 진행하다 중단된 목표 불러오기를 했을 때(비정상 종료시를 위해 만든 기능이다.)와 버튼을 직접 사용자가 눌렀을 때 이다. 
> instance by lazy로 인스턴스한 goalGuide는 (loop를 도는 루틴의 주체이다.) collect 특성상, 앞 루틴이 실행된 후에야 나중 루틴을 수행한다. 동시에 병렬로 수행하지 않는다. 

그럼 어디가 문제였냐? 굳이 애니메이션에서 중단되는 문제는 .ensureActive때문이 아닌(즉, 다른 외부에서 의도적으로 해당 '코루틴'을 중지시킨 경우가 아니었다.), 바로 애니메이션 실행 코드 자체에 있었다.
```
//40초 동안 사용자의 터치가 없다면, 혹은 사용자가 터치한다면
                withTimeoutOrNull(40000L) {
                    LogData.logFlow.first { LogData.isTouched }
                }
                //넘어가서 실행
                animationPlayer.removeGuide(windowManager)
```
어쩌면 당연한게, 로직 자체가 사용자가 가이드 애니메이션을 따라 터치를 했을 경우, 애니메이션은 즉시 종료되도록 했다.
- 올바른 좌표인지 아닌지는 확인하지 않는다. 
- 올바른 좌표가 아닐 때 막는 기능, 예를 들어 해당 좌표빼고 모두 블락하는 기능은 (애초에 그럴 수 있나?) 뭔가 기분이 찜찜해서 안 넣었다. 가이드라기보다는 무언가 강제하는 체벌같다. 

그럼 사용자가 터치를 했었느냐? 나는 분명 터치를 하지 않았고, 로그만 봐도 정말 순식간에 터치가 이루어졌다. 

그렇다. 코드 문제이다. 접근성 서비스에서 사용자 터치를 감지했기 때문인데, 이건 내 코드에 오류가 있어서 그렇다. 

앞에서 말했듯 특정 view들은 사용자 터치를 반응하지 않고, 따라서 나는 TYPE_WINDOWS_CHANGED(안드로이드 시스템에서 window 단위의 변화가 있을 때) 플래그가 있을 때면 isTouched 라는 터치 플래그를 true로 바꾸게끔 설계했다. 범인을 찾았다.

### 필요한 상태를 정의해보자

내 앱에서, 접근성 이벤트는 Ui 상태(이하 '상태')의 갱신인데, 이때마다 collect{} 루틴을 돌게할 것이다. 

그렇다고 이벤트가 쏟아지는 (나름 고심해서 선택한다고는 했지만, 여전히 많다. 아직 수정해야할 것이다.) 상황에서 이벤트 감지 범위를 줄이자니 팝업창이라던가, 뭔가 특별한(접근성 서비스 대부분을 발생시키지 않는 view- 예를 들면 터치를 절대 감지하지 않는다.) view들에 대해 감지를 못하는 경우가 있어 어느정도 감내해야만 했다. 

물론 이대론 그냥 둘 수 없다. 지금 원하는 수정 방향은 기존에 진행 중이던 대답과 중복이면 아예 새로 시작될 루틴을 취소해 버리는 경우이다. 이 방법은 루틴 중단 시점에 따라 나눌 수 있을 것 같다.
1. 상태 갱신 이전, 이벤트 감지시점에서부터(접근성 서비스 모듈), 즉 접근성 서비스의 이벤트를 보고, 대답이 중복일 것 같은 경우인지 판단한다. 
2. 상태 갱신 이후, 앞선 루틴이 있으면, collect{}에 앞선 루틴이 모두 실행되기를 기다리는 중에 취소를 판단한다.
3. 대답을 확인하고 코루틴을 중지시키거나 화면에 반영하지 않는다.

처음 든 생각으로서는, 3번은 너무 느리다. 어느세월에 다시 LLM에게 쿼리를 보내고, 추론시간을 기다려 완성된 응답을 받아오지? 이미 `accessibility_service_config.xml`파일에서 `android:notificationTimeout="300"`(해당 flag 값을 0.3초)으로 설정해뒀다. 
게다가 collect에서 `.debounce(3000)` 메소드로 3초간 stateFlow에서 갱신되는 상태 값들의 반영을 지연시키는데, 이 효과는 android:....="300" 보다 더 크다. 

우선은 1번과 2번을 먼저 검토해보자.

1번과 2번 중에선 1번이 여러모로 더 좋은 것 같다. '미리' 판단함으로써 사전에 불필요한 진행결과를 막는다는 이점에 더해, 서비스의 오버헤드를(성능 낭비를!) 빠르게 막을 수 있다.

그렇게 1번을 골랐으니, 이번에는 '상태의 중요성'을 어떻게 판단할 것인가가 관건이다. 여기서 상태의 중요성은 "이 상태 변화가 중복된 대답을 받아오는가?"가 기준이 되겠다. 그래도 여전히 추상적이다. 더 구체적으로 판단해보자.
- 상태란 무엇일까? ui 스냅샷 결과이다.
- 상태의 갱신은? '사용자의 화면 상호작용' 뒤에 나타난 'ui 스냅샷의 변화'가 반영된, 새로운 ui 스냅샷 결과이다.
- 상태의 변화가 반영된 새로운 상태가, 이전 상태와 동일한 LLM 대답(지시사항)을 내놓을지는 어떻게 판단하는가?
> LLM의 대답은 지시사항(한 줄짜리 지시문)을 제외하고, '행동'과 'ui 좌표'로 제한했다. 
> - 여기서 다시 행동은 행동 list 중에서 하나를 고르는 형식이다. (하네스도 결국 이런 제한을 두는 방식이 아닌가?)
> - ui 좌표는 LLM에게 보내는 ui 정보에 같이 명시됐다.

지금 ui 스냅샷으로 LLM에 넘겨주는 정보는, 다음과 같다. (제미나이가 잘 정리해주었기에 역시나 복붙을 한다.)

```
1. 수집하고 있는 정보 분석

**가. 텍스트 및 의미적 식별 정보 (Semantic Information)**

사용자가 화면에서 읽을 수 있는 글자나 해당 요소가 무엇인지 설명하는 정보를 수집합니다.

  - Text: 뷰에 표시된 실제 텍스트입니다.
  - Content Description: 이미지 버튼처럼 텍스트는 없지만 접근성 지원을 위해 설정된 설명 문구입니다.
  - View ID: 개발자가 부여한 리소스 ID(`viewIdResourceName`)를 통해 해당 요소의 기능적 역할을 파악합니다.

**나. 사용자 상호작용 및 상태 정보 (State & Actionability)**

해당 요소가 사용자와 어떻게 상호작용할 수 있는지, 현재 어떤 상태인지를 파악합니다.

  - Actionability: 클릭 가능(`isClickable`), 포커스 가능(`isFocusable`), 수정 가능(`isEditable`) 여부 등을 통해 조작 가능성을 판단합니다.
  - State: 현재 선택되었는지(`isSelected`), 스크롤이 가능한지(`isScrollable`) 등의 상태를 수집합니다.

**다. 시각적 위치 및 가시성 정보 (Spatial Information)**

요소가 화면 어디에 위치하며, 사용자에게 실제로 보이는지를 파악합니다.

  - Bounds in Screen: 화면 전체 좌표계에서의 절대적 위치(상하좌우 좌표)를 수집합니다.
  - Visibility: 현재 사용자에게 실제로 보이는 요소(`isVisibleToUser`)만 필터링하여 유효한 데이터만 남깁니다.

2. 판단 근거 및 공식 문서 링크

각 수집 항목에 대한 상세 설명은 안드로이드 공식 개발자 문서인 [AccessibilityNodeInfo](https://developer.android.com/reference/android/view/accessibility/AccessibilityNodeInfo)에서 확인할 수 있습니다.
```

원래는 여기에 위계 정보까지 추가했었는데, \t 키로 구분하니 너무 과도한 토큰 낭비같아 모두 없앴다. 현재는 ' '로 한 칸씩 트리구조의 깊이를 표시하고 있다. 그리고 실제로 성능차이도 빠르면 빨랐지, 올바른 ui 선택률이 나빠졌다고는 느끼지 못했다.

코드도 일부 가져오자면,
```kotlin
if ((isMeaningful || isActionable) && nodeInfo.isVisibleToUser) {
            val bounds = Rect()
            nodeInfo.getBoundsInScreen(bounds)

            // 3. 상태 정보 수집
            val states = mutableListOf<String>()
            if (nodeInfo.isClickable) states.add("Clickable")
            if (nodeInfo.isScrollable) states.add("Scrollable")
            if (nodeInfo.isEditable) states.add("Editable")
            if (nodeInfo.isSelected) states.add("Selected")
            if (nodeInfo.isCheckable) states.add("(Checkable)")
            val checkedState = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.BAKLAVA) {
                nodeInfo.getChecked()
            } else {
                if (nodeInfo.isChecked) 1 else 0
            }
            when (checkedState) {
                AccessibilityNodeInfo.CHECKED_STATE_FALSE -> states.add("not checked")
                AccessibilityNodeInfo.CHECKED_STATE_TRUE -> states.add("Checked")         // CHECKED
                AccessibilityNodeInfo.CHECKED_STATE_PARTIAL -> states.add("PartiallyChecked") // PARTIAL
            }

            val stateString = if (states.isNotEmpty()) "[${states.joinToString(",")}] " else ""
            val resourceId = nodeInfo.viewIdResourceName ?: "No ID"
            val className = nodeInfo.className
            // 4. 한 줄로 합침
            logBuilder.append("-(class: $className) $stateString\"$label\" (ID: $resourceId): $bounds\n")
        }
```
이렇게 정보를 String에 계속 추가하면서 노드들을 순회하는 방식이다. 

아무튼 ui 스냅샷(LLM의 판단)에 들어갈 정보는 
1. 사용자에게 보이고
2. 조작이 가능하거나, 텍스트가 있을 때, 혹은 노드 자체에 설명이 달려있을 때
3. 해당 정보들을 추가한다. 단, 한 칸 들여쓰기로 깊이를 나타낸다.
4. 여기에 해당 노드의 클래스(Button, TextView 등등)을 추가하고, ID도 추가하며, 설명(이 있을 경우)도 추가한다.
5. 끝으로 좌표위치를 추가한다.

여기서 LLM의 행동을 이끌어낼 수 있는 요소들이 무엇일지 정확히 골라내기란 어려운 일이다. 사용자의 클릭으로 작은 구석의 view 안 text가 한 글자 바뀔 수도 있다. 아..., 그렇다면 1,2번을 포기하고 다시 3번으로 돌아오는 경우를 생각해보자.

앞선 애니메이션을 취소하는 과정은 내 앱의 코드 안에서 애니메이션을 실행하는 과정 바로 직전이다. (항상 애니메이션을 실행전에 앞선 애니메이션은 초기화하는 코드이다.) 

그렇다면 기존 애니메이션이 끊기고(자원 반환에 생각보다 시간이 걸렸었다.) 새로운 애니메이션을 넣기전에, 그러니까 LLM의 응답을 받아온 직후에 이전과 LLM의 판단을 비교 후, 삭제하는 것이 좋을 것이다. 

LLM의 판단을 비교할 요소는 이미 고민한 부분과 일치한다. 바로 '행동'과 '좌표'이다. 

이것들이 일종의 signature가 되는 것이다. 단, 같은 위치에 계속해서 버튼이 생기거나(업데이트의 '다음'버튼 등) 같은 image view를 계속 슬라이드 해야하는 등의 경우가 있을 수 있겠다. 

따라서 시그니처가 겹치는 경우를 줄이기 위해, 비교할 요소를 조금 더 추가해야겠다. 
- view id? 없는 경우가 너무 많았다.
- view class? 너무 포괄적이라 구분하는 용도의 요소로는 힘들 것 같다.
- view 안 text? view 안 text는 또 너무 구체적이라 이 요소의 변화 자체가 ui 스냅샷 상태 변화를 일으켰을 수도 있다. 조금 더 포괄하면 좋겠다.
- 현재목표? 최종 목표에 비해 매 ui 상황마다 변할 수도 있다.
- 지시사항? 여태것 footprint로서 LLM이 현재 '반복 패턴'을 지시하고 있는지 판단하는 근거가 된다. 

현재목표나 지시사항이 될 것 같은데, 여기서는 기존에 있던 지시사항을 사용하기로 했다.
> 따라서, **지시사항**, **좌표**, **행동**을 묶어서 `GuideSignature`라고 하기로 했다.
> - 이 GuideSignature를 보고 LLM의 응답이 중복인지 판단, 애니메이션 로직으로 넘어가서 이전 애니메이션을 취소하기 전, 미리 return 한다.

말 나온 김에 data 타입으로 지정해놓자.
```kotlin
data class GuideSignature(
    val instruction: String,
    val coordinates: List<Int>,
    val action: String
)
```

그리고 이곳저곳에서 활용해본다. 아래에 `.toGuideSignature()`은 별도의 확장함수`Extension Function`을 추가한 것이다. 기존 클래스(String)의 소스 코드를 수정하지 않고도 새로운 메서드를 추가했다. 코틀린 기능이랜다.

```kotlin

//(GoalGuide, 즉 collect{}를 통해 일종의 loop job(루틴) 코드 안)
//...
//지시사항, ui 요소 좌표, 행동을 signature로 이전과 비교.
                val (instructionText, coordinateNumbers, guidedAction) = StringEditor.extractSignature(responseText)
                if(GuideSignature(instructionText.trim(), coordinateNumbers, guidedAction.trim())==
                (PromptBuffer.getFootprint(PromptBuffer.FootprintViewOptions.LAST_ONLY).toGuideSignature())){ // 가독성 최악이다. 하지만 일단은 넘어가자. 일단은....
                    log("같은 가이드 시그니처 발생. return.")
                    return@collect
                }
                //...
```

이렇게 두 개의 시그니처를 비교한다. 전자는 방금 받아온 LLM의 응답을, 후자는 footprint에서 나온 것이다.

여기에 더해, 접근성 서비스 자체에서부터 resouceId를 키로 두고, event가 일정 시간 동안 같은 키에서 발생하면 쿨타임을 두는 것이다.

처음 한 번 발생시에는 저장을 해두고, 두 번째 부터는 일정 시간동안 return 하도록 설정해둔다. 물론, 실제로 ui가 변화한 것일 수도 있으므로, 대략 0.5~6초, 혹은 <1초 간의 쿨을 두도록 한다.
- 대략 5개 정도의 entry를 가진 리스트를 생각하다가, 5개가 넘으면 덮어씌우는 원형 리스트가 있으면 좋겠다고 생각했다.
- 젬나이한테 물어보니까 이럴 때 쓰라고 있는 linked hash map이 있댄다. linked list에, hash를 써서 O(1)이고, map<k,v>까지 있다. 
- 심지어는 정렬기능도 있다고 했다. 인자는 아래와 같다.
> - initialCapacity: 초기 entry 최대 갯수이다.
> - loadFactor: default가 0.75(75%)이다. 75%가 차면 배열 크기를 키우겠다는 얘기이다.
> - accessOrder: 기본값은 false. FIFO인데, true로 할 시, LRU, 즉 Least Recently Used가 된다. 

```kotlin
companion object {
        ...
        // 동일한 ID에 대해 1초 동안 무시.
        private const val VIEW_DEBOUNCE_MILLIS = 1000L
        private const val MAX_ENTRIES = 5
    }
    // 특정 resourceID별로 마지막 처리 시간을 저장 (ID : Timestamp)
    // 고정된 크기를 가진 원형 버퍼 형태의 맵
    private val viewEventTimestamps = object : LinkedHashMap<String, Long>(MAX_ENTRIES, 0.75f, true) {
        override fun removeEldestEntry(eldest: MutableMap.MutableEntry<String, Long>?): Boolean {
            return size > MAX_ENTRIES
        }
    }
    // 마지막으로 removeEldestEntry를 override해서 정한 갯수를 넘으면 가장 앞의 값(여기선 LinkedHasgMap의 세 번째 인자를 treu로 /////했기에, LAFO가 된다.)
    override fun removeEldestEntry(eldest: MutableMap.MutableEntry<String, Long>?): Boolean {
        return size > MAX_ENTRIES
    }
```
요렇게 함으로써, 앞에서 봤던 isTouched가 인식돼 애니메이션이 강제종료되는 것을 조금은 줄일 수 있을 것이다.

### 목표달성시 목표달성 문구가 포함된 말풍선이 보이지 않는 문제

그리고 또, 목표달성시 "목표달성! + 간단한 설명"를 사용자에게 보여주는 것 또한 사용자 경험 측면에서 몹시 중요하다 생각했는데, 테스트를 해보니 '목표달성' 반환 뒤, 뒤이어 들어오는 collect{}루틴이 (loopJob이라 명하고 있다.) 실행 중인 말풍선을 취소해버려서 "목표달성!..."이 써있는 말풍선이 화면에 표시되지 않는 현상이 있었다. 

역시나 범인으로 의심되는 부분은 isTouched가 들어오면 말풍선을 취소하고, 각종 job 객체를 cancel()해버리는 부분이다.

우선 GoalGuide안 collect{} 내 루틴(loopJob 객체의 코루틴으로 실행하는 부분이다.)에서도 막아준다. 코드를 그새 조금 수정했는데, `PromptBuffer....toGuideSignature()`까지해서 받아오는 걸 고쳤다.
> 사실 뭐 이래저래 더 고친게 있다. 
> - toGuideSignature를 하려니 처음에는 쉼표 단위로 파싱했었는데,(Footprint안에서 \n으로 구분된 한 줄 마다 `$instruction,$coordinates,$action`으로 넣어주고 있었다.) 문제는 저 coordinates가 List<Int>라 쉼표 구분이다. 급히 구분 문자를 `"\t"`문자로 바꿔주었다.

아무튼 그렇게 받아온 footprint 제일 마지막줄(가장 최근) signature와 지금 LLM에 받아온 signature를 비교한다.

```kotlin
(GoalGuide 안 collect{} 내 코드)
...
                 //지시사항, ui 요소 좌표, 행동을 signature로 이전과 비교.
                val (instructionText, coordinateNumbers, guidedAction) = StringEditor.extractSignature(responseText)
                val lastGuideSignature = PromptBuffer.getFootprint(PromptBuffer.FootprintViewOptions.LAST_ONLY).toGuideSignature()
                if(GuideSignature(instructionText.trim(), coordinateNumbers, guidedAction.trim())==
                    lastGuideSignature){
                    log("같은 가이드 시그니처 발생. return.")
                    return@collect
                }
                //앞선 루프가 "목표달성!"시에도 return
                if(lastGuideSignature.instruction.contains("목표달성")) return@collect
                ...
```

다음으로 colelct{} 후 overlayService 에서 실행하는 (이곳을 사실상 메인으로 하고 있고, 여기서 모든 ui 작업을 실행한다.) 애니메이션 실행 루틴을 손본다.
```kotlin
 override suspend fun preprocessResponse(resText: String?) {
        if (resText == null) return
        if (resText.contains("목표달성")) {
            withContext(Dispatchers.Main + NonCancellable) {
                LogData.isMicUsed.set(false)
    
                val aiMessage = StringEditor.extractTargetLine(resText, "목표달성!")
                val finalMessage = if (aiMessage.isNotBlank()) {
                    "목표달성! ${aiMessage.trim()}"  // <-- 이렇게 추가
                } else {
                    "목표달성! 성공했어요. 더 궁금한 게 있다면 저를 눌러주세요."
                }
                // 목표 달성시 저장하던 지난 목표 지우기
                clearLastGoal()
                PromptBuffer.clearAll()
    
                // 루프 정지
                loopJob?.cancel()
                loopJob = null
    
                // 말풍선 띄우기
                showSpeechBubble(finalMessage, 5000L)
            }
            return
        }else {
```
제일 아래 else { ...  부터가 애니메이션을 화면에 띄우는(재생하는?) 부분이고, if문에서는 목표달성이라는 문구가 LLM의 응답에 들어있는지 확인 후, 별도의 루틴을 수행한다. 

눈에 띄는 점은 //말풍선 띄우기의 말풍선 실행 부분이 `loopJob?.cancel()`뒤에 있다는 점이고, 또 하나는 제일 위 쪽에, 두 번째 if...줄 바로 다음줄, `.set(false)` 부분이다.

일단, 말풍선은 `suspend` 함수가 아니다. 안에서 별도의 serviceScopeMain 이라는 별도의 CoroutineScope를 만들고 있다. 
- 심지어는 이 serviceScopeMain이라는 CoroutineScope는, loopJob의 CoroutineScope와 같다. 그리고 버튼을 눌렀을 때 실행되는 CoroutineScope와도 같다. 

이 말은, 부모-자식 관계가 아닌, **형제 관계**란 뜻이다. 즉, 한 쪽을 취소한다고, 다른 쪽의 코루틴에 영향을 줄 수 없다. (물론 Job 객체를 잘 받아와서 cancel() 등의 메서드로 취소할 수는 있다. 여기선 생명주기를 얘기한 것이다.)

따라서, cancel()로 자원 반납을 시켜놓고, 해당 코루틴이 잘 마무리되든 말든 말풍선은 그대로 실행해버린다는 뜻이다. 반응성도 좋다.
- 여기서 반응성이라 함은, `cancelAndJoin()` 메서드와 비교해서 한 말이다. 해당 메서드는 자원 반납과 같은 마무리 루틴을 모두 기다린 후에야 다음 줄 코드로 넘어간다.

그리고 두 번째! .set(false) 부분으로 넘어가자!

#### 원자적 수행으로 race condition을 막아보자

코틀린에는 간단한 변수 값 변경 / 확인에 쓰이는 원자적 연산이 있다.
> 원자적 연산이란, 하드웨어(H/W) 수준에서 지원하는 명령어 단위이다. 이 원자적 연산을 하드웨어가 수행하는 사이에 다른 명령어가 끼어들 틈이 없다. 
> - 더이상 나눌 수 없는, (틈이 없는?) 명령어라 '원자적(Atomic)'이 붙은 것 같다.

AtomicBoolean()과 AtomicInteger()가 그 예이다.

```kotlin
import java.util.concurrent.atomic.AtomicInteger
import java.util.concurrent.atomic.AtomicBoolean
```
으로 import해서 사용할 수 있다. 

```kotlin
AtomicBoolean 객체에 대해서

set() 원하는 AtomicBoolean 타입 값으로 무조건 지정한다.

get() 호출 시점의 AtomicBoolean 타입의 값을 반환한다.

comapareAndSet(a, b) a 타입이면 b로 바꾼다. 성공시 true를 반환한다. (실패시 false를 반환한다.)

compareAndExchange(a, b) a타입이면 b로 바꾼다. 단, 성공 여부와 관계없이 바꾸기 직전 값을 반환한다.
```

기존의 LogData의 flag로 쓰이는 멤버 변수들(isTouched, isMicUsed)를 원자적 연산으로 바꾸고 / 검사하기로 했다.
```kotlin
LogData.isTouched = AtomicBoolean(false) // 로 초기화를 하고, 후에

// 외부 서비스
if(LogData.isTouched.compareAndSet(false, true)){
    //... 실행 코드
    LogData.isTouched.set(false) // 문닫고 나온다.
} 

//로 써주거나, 혹은,

if(조건문을 만족하면){
    LogData.isTouched.compareAndSet(false, true )//이렇게 false였을 때 true로 바꿔준다.
}
```
collect{}의 첫 줄에서 isTouched와 isMicUsed가 &&(논리적 and 연산)으로 필터링 중이니, isTouched나 isMicUsed가 반드시 true여야한다.

하지만 목표달성을 인식하자마자 false로 바꿔버려서 그 사이에 새로운 ui 상태가 업데이트돼 collect{...}가 실행되더라도 첫 줄에서 막히게 된다. 사용자 이벤트가 들어와 isTouched는 true가 될지라도, isMicUsed는 사용자가 오버레이 아이콘 버튼을 눌렀을 때, 혹은 저장됐던 목표를 불러올 때에만 true로 바뀌게 된다. (이것들 역시 set()이나 compareAndSet()으로 바꾼다.)

끝! 일단은, 잘 돌아간다.... 또 문제가 나오면 수정해야겠지

...To Be Continue...