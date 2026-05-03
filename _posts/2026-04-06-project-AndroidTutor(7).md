---
layout: post
title:  "[project]AndroidTutor(5) 안드로이드 접근성 서비스에 관한 고찰"
author: kau-newbie
categories: [ android, project, accessibilityService, 접근성서비스]
image: assets/images/prj-androidtutor-basicmodel.jpg
---

[접근성 이벤트를 알아보았다.](https://developer.android.com/guide/topics/ui/accessibility?_gl=1*1x53esw*_up*MQ..*_ga*MTQ1MTE1MDg5MS4xNzc3NzkxODMy*_ga_6HH9YJMN9M*czE3Nzc3OTE4MzEkbzEkZzAkdDE3Nzc3OTIwNDkkajYwJGwwJGg5NzQ2MjQ1NTA.&hl=ko)

# 안드로이드 접근성 서비스에 관한 고찰

일단 지금 개발중인 앱은 안드로이드 시스템에서 접근성 서비스 이벤트를 쏘아주면(사용자가 물리적으로 화면을 조작했으면 무조건 ui 변경이 일어날 것이라는 가정.), 앱 내부 로직에 의해 LLM에게 다음 행동을 지시받는 원리이다.

여러 문제가 있겠지만, 그 중에서도 접근성 서비스 필터링 문제는 여간 중요하지 않을 수 없다.

접근성 이벤트를 너무 세세하게 받으면, 화면상의 광고창이 바뀌거나, 시계와 같이 주기적으로 바뀌는 text가 변경되거나, 기타 등등의 모든 일들에 대해 LLM에게 request를 보내게 된다.

지금까지는 주먹구구 식으로 이 접근성 이벤트들을 막아본 경향이 없지 않아 있다.

왜냐하면, 근본적으로 "사용자의 터치가 있으면 --> 화면이 의미있게 변할 것이다"는 가정이 항상 성립하지 않기 때문이다.
- 여기서 '의미 있다'라는 말은, 사용자가 원하는 목표(예: 문자 메시지를 00에게 보내기)를 달성하는 과정에서 도움이 되지 않는 정보(예: 시계 text가 a.m. 10:00 --> a.m. 10:01로 변경됨.)를 뜻한다.

예를 들자면, 새로운 창이 뜨는 순간에도 단순히 TYPE_WINDOW_STATE_CHANGED라는 이벤트만 발생하는 게 아닌, 갖가지 text들 (예를 들면, 아이콘 밑의 앱 이름들이 로딩됨에 따라 생김)이 변경/발생함에 따라 TYPE_WINDOW_CONTENT_CHANGED가 우후죽순 발생한다. 

그렇다고 이 TYPE_WINDOW_CONTEnT_CHANGED를 막을 순 없는 게, 삼성 UI 런쳐 같은 경우에는 앱 드로어(앱 목록)을 열 때 TYPE_WINDOW_STATE_CHANGED를 발생시키지 않는다.
- 지금까지는 그렇게 알고 있다.
- 혹시 모르는 게, 이 접근성 이벤트가 너무 우후죽순 쏟아지다 보니 시스템에서 미처 보내지 못한 건 아닌가, 하는 1%의 의심도 있다.

해서 지금까진 parsing을 통해 특정 문자열, 예를 들어 'home'이라던가, 'launcher'라는 이름이 들어간 class, 혹은 view id에 대해서는 TYPE_CONTENT_CHANGED를 허용하는 식으로 그때그때 상황을 봐 가며 필터링을 추가하는 중이다.
- 여기서 '허용'이란, 앱 로직 상 LLM에게 매번 api를 (request)보내는 것을 방지하기 위해 새운 flag 변수를 true로 바꾼다는 의미이다.

다만, 여전히 삼성 UI 런쳐에 대해 의도한 대로 동작하지 못 하는 상황들이 발생하기에, 이번 기회에 접근성 서비스를 제대로 공부해 보려고 한다.

## 필터링 문제

사실 세팅은 워낙 [안드로이드 공식문서](https://developer.android.com/guide/topics/ui/accessibility/service?hl=ko)에 잘 나와있다.

중요한 건, 역시나 필터링 관련된 부분들이다.

먼저, AccessibilityService를 상속받은 서비스 하나를 만들고, onAccessibilityEvent(event: AccessibilityEvent)도 구현하고...,

manifest에서

```xml

<application>
  <service android:name=".accessibility.MyAccessibilityService"
  ...

```

도 구현하고...,

`res/xml/accessibility_service_config.xml에서 구성 파일을 만듭니다. 이 파일은 서비스에서 처리하는 이벤트와 제공하는 피드백을 정의합니다.` 여기서부터이다.

공식문서 상의 xml파일 예시는 다음과 같다.

```xml

<accessibility-service xmlns:android="http://schemas.android.com/apk/res/android"
    android:description="@string/accessibility_service_description"
    android:accessibilityEventTypes="typeAllMask"
    android:accessibilityFlags="flagDefault|flagRequestFingerprintGestures|flagRequestAccessibilityButton"
    android:accessibilityFeedbackType="feedbackSpoken"
    android:notificationTimeout="100"
    android:canRetrieveWindowContent="true"
    android:canPerformGestures="true"
    android:settingsActivity="com.example.android.apis.accessibility.ServiceSettingsActivity" />

```

두 번째 줄의 `android:accessibilityEventTypes="typeAllMask"`를 먼저 살펴보자.

참고: typeAllMask를 사용하면 시스템에서 전체 OS에서 발생하는 모든 접근성 이벤트를 애플리케이션에 알리도록 강제하므로 리소스가 많이 사용될 수 있습니다. 라는 경고문구까지 친절히 달려있다.

하지만 범용성 앱을 위해선 typeAllMask를 권장한다고 한다. 옵션들을 더 살펴보자.
[옵션 참고](https://developer.android.com/reference/kotlin/android/accessibilityservice/AccessibilityServiceInfo?_gl=1*193qfvs*_up*MQ..*_ga*MTQ1MTE1MDg5MS4xNzc3NzkxODMy*_ga_6HH9YJMN9M*czE3Nzc3OTE4MzEkbzEkZzAkdDE3Nzc3OTE4MzEkajYwJGwwJGg5NzQ2MjQ1NTA.#attr_AccessibilityServiceInfo)

이 옵션들의 정체는 `android.view.accessibility.AccessibilityEvent`에 정의된 상수값들이다. 
> class AccessibilityEvent : AccessibilityRecord, Parcelable

앗, accessibilityRecord라는 클래스를 상속받으면서 parcelable은 구현하고 있다. 꼬리에 꼬리를 물고 이어지는 느낌이지만, 일단 파고들어보자.

#### 1. parcelable

`android.os.Parcelable`

말그대로 parcel + -able이다. `parcel`형태로 변경할 수 있다는 건데, java의 Serializable보다 효율적인 객체의 직렬화 통신이 가능하게 한다고 한다.

이게 왜 필요한지 젬나이에게 물어봤는데, 당연한 얘기지만 서로 다른 프로세스들은 서로 다른 메모리 공간을 보고 있다. (프로세스 간 mem 격리는 OS 수업에서 배운 바 있다.) 따라서 주소를 넘겨줘도 서로의 객체를 사용할 순 없다. 때문에 객체를 직렬화해 다른 프로세스에게 넘겨주게 되는데, 이 전송할 때 parcel이라는 container에 맞게 담아 보낸 뒤, 받는 쪽 프로세스에서 조립한다고 한다.
- 여기서 프로세스들은 각각 앱과 시스템(접근성 서비스를 발생시키는 주체)이 되겠다.

안드로이드 문서에 보면 parcel에 대해 다음과 같은 설명이 있다.

Parcel is not a general-purpose serialization mechanism. This class (and the corresponding Parcelable API for placing arbitrary objects into a Parcel) is designed as a high-performance IPC transport. 
> IPC transport를 위해 특별하게 사용되는, 성능 좋은 클래스라고 한다.
> - IPC transport란, IPC(Inter-Process Communication) Transport, 즉 프로세스간 통신 통로이다.
> - **대표적인 IPC transport가 Binder이다!**
> - IPC transport외에도 프로세스간 통신 방식은 다양하다고 한다.
> - 다른 방식의 Transport들
>    - 안드로이드 바인더 외에도 컴퓨팅 세계에는 다양한 IPC Transport가 존재.
>    - Sockets: 네트워크 통신과 비슷하게 로컬 프로세스끼리 통신하는 방식 (주로 유닉스 도메인 소켓).
>    - Shared Memory: 두 프로세스가 똑같은 메모리 방을 공유해서 쓰는 방식 (속도는 제일 빠르지만 동기화가 매우 어려움).
>    - Message Queues: 운영체제가 관리하는 메시지 보관함에 넣고 빼는 방식.

Container for a message (data and object references) that can be sent through an IBinder.
> IBinder를 통해 보내는 메시지를 담는 객체이다. 
> - IBinder란, interface로, 이 IBinder를 구현한 구현체가 Binder이다.

*| Binder?*

프로세스a가 프로세스b에게 객체를 보낼 때, 보통은 message passing 방식이기 때문에, 커널을 통해 커널 소유의 메모리 공간을 사용한다.

우체통에다 객체 넣어두면, b가 받아가는 방식이다. 이때 두 번의 copy가 발생하는데, Binder는 여기서 더 발전해 'MMap'이라는 방식을 사용한다.

프로세스 b의 메모리 공간의 일부가 커널의 일부 메모리 공간을 가리키도록 매핑하는 것이다. 그렇게 되면 b 입장에서는 본인의 메모리를 읽는 셈이 된다.

다시 돌아와서, 

#### 2. accessibilityRecord

Represents a record in an AccessibilityEvent and contains information about state change of its source android.view.View. 
> 말그대로 접근성 이벤트를 발생시킨 view의 상태 변화 정보를 담고 있는 기록물이다.
>
> 일단 한 번 이벤트가 발생하고 record의 구현체가 생성되면, 변하지 않는(immutable)다고 한다.

메서드 정의에는 
- getClassName()
- getSource()
- isChecked()

같은 친숙한 메서드들도 보여 반가웠다. 다만, 새로 메서드들을 발견했는데,

`getDisplayId()`와 `getBeforeText()`이다.

먼저, getBeforeText()는 기존 뷰에 있던 text를 보여준다고 한다.

이 말인 즉슨, 이벤트가 발생하면, 어떤 뷰에 지금 (바뀐) text 뿐만 아니라, 비교를 위해 이전 text까지 같이 보여준단 뜻이다.
- 사용자가 텍스트를 작성하다 수정했을 때, 어떤 글자를 지웠는지 알아볼 수 있는 무시무시하고 유용한 기능이었다.

두 번째로는 *DisplayId*인데, 이 display 라는 단위가 생소했다. 그래서 젬나이한테 물어봤다.

*코드상의 위치와 관계*

안드로이드 UI 계층 구조를 큰 범위에서 좁은 범위로 나열하면 다음과 같습니다:

- Display: 물리적/논리적 화면 (Display ID)

- Window: 하나의 화면 안에 떠 있는 창 (Window ID)

- View Hierarchy: 창 내부의 버튼, 텍스트 등 (View ID / AccessibilityNodeInfo)

실제로 display라는 다중 창을 지원한다고 한다. 왜냐하면 최근 폴더블 폰 같은 여러개의 물리적/논리적 디스플레이를 가진 기기가 많아졌기 때문이라고 한다.


#### AccessibilityEvent

아무튼 다시 AccessibilityEvent로 돌아오자.

This class represents accessibility events that are sent by the system when something notable happens in the user interface. For example, when a android.widget.Button is clicked, a android.view.View is focused, etc.
> 잘 알다시피, 사용자의 interface 조작으로 주목할만한 이벤트가 일어났을 때, 해당 이벤트를 의미한다. 예: 버튼 클릭 등등

The main purpose of an accessibility event is to communicate changes in the UI to an android.accessibilityservice.AccessibilityService. If needed, the service may then inspect the user interface by examining the View hierarchy through the event's source, as represented by a tree of AccessibilityNodeInfos (snapshot of a View state) which can be used for exploring the window content. 
> 접근성 이벤트는 접근성 서비스에 ui 변화를 감지해서 알리는 목적이 있다. 여기서 view state의 snapshot이라는 표현이 나오는데, 정확히 우리 앱의 UIStateFlow에 담길 state에 해당한다.
> - accessibiltiyNodeInfos의 트리 구조 정보를 스냅샷으로 담을 것이다.

For each event type there is a corresponding constant defined in this class. Follows a specification of the event types and their associated properties:
> 각 종류의 이벤트들은 상수로 정의돼있다.

```kotlin
getEventType()
```

으로 얻을 수 있는 event type들에는 다음과 같은 것들이 있다.

[event types](https://developer.android.com/reference/kotlin/android/view/accessibility/AccessibilityEvent?_gl=1*1qm1djr*_up*MQ..*_ga*MTQ1MTE1MDg5MS4xNzc3NzkxODMy*_ga_6HH9YJMN9M*czE3Nzc3OTE4MzEkbzEkZzAkdDE3Nzc3OTE5ODEkajYwJGwwJGg5NzQ2MjQ1NTA.#summary)

TYPE_VIEW_로 시작하는 action들은 모두 사용자가 view를 조작했을 때 발생하는 이벤트들이다.
- 우리 앱에서는 isTouched라는 flag 변수를 true로 바꾸는 데 사용하고 있다.

사용자 행동 이벤트를 제외하고

여기서 주목할 것들은 경험상 아래의 세 가지 정도이다.
1. TYPE_WINDOW_STATE_CHANGED
- 새로운 pane의 생성/삭제
- pane의 title 변경
    - window가 앱 화면 전체라면, pane은 window를 더 작은 영역 한 개 이상으로 나눈 것들.

2. TYPE_WINDOW_CONTENT_CHANGED
- 노드의(view) checked 정보가 변했을 때 (체크 표시가 활성화)
- content description이 바뀌었을 때 - 예를 들어, 삼성 ui는 홈화면 페이지 수를 1/2 2/2 와 같이 description 해둔다.
- 뷰 요소를 드래그 시작/중단/놓았을 때
- 뷰 요소에 interact가 가능/불가 해졌을 때
- 드롭 다운같이 펼치고/ 접을 수 있는 뷰가 펼쳐지고/ 접힐 때
- 에러 메시지를 반환할 때
- view의 text가 변경됐을 때
- veiw의 자식 노드가 변경됐을 때

3. TYPE_WINDOWS_CANGED
- The window's AccessibilityWindowInfo.isActive() changed.
    - An active window is the one the user is currently touching or the window has input focus and the user is not touching any window.
- window가 추가/제거 됐을 때
- his event implies the window's region changed. It's also possible that region changed but bounds doesn't.
> 창의 위치가 변경됐을 때
- window의 자식(하위 ui요소들)이 변경사항이 있을 때
    - TYPE_WINDOW_CONTENT_CANGED도 바뀐다.
- input focus가 변할 때 (id 박스에서 id적고 password 칸으로 옮겨갈 때)
- `WINDOWS_CHANGE_LAYER` z-order가 바뀌었을 때! 이건 정말 유용해 보인다.
- 윈도우의 부모가 바뀔 때: 이건 젬나이에게 물어봤더니, 다음과 같은 예시가 있다고 했다.
> 다이얼로그가 다른 윈도우에 붙을 때
>
> 예: 앱 내에서 다이얼로그가 처음에는 Activity A의 윈도우에 붙어 있다가, Activity B로 전환되면서 부모 윈도우가 바뀌는 경우.
>
> 팝업 윈도우가 재배치될 때
>
> 예: 드롭다운 메뉴나 툴팁 같은 팝업 윈도우가 원래는 특정 뷰(Window)의 자식이었는데, UI 구조 변경으로 다른 부모 윈도우에 속하게 되는 경우.
>
> 멀티윈도우 모드에서 창 이동
>
> 예: 분할 화면 모드에서 앱 창을 드래그해서 다른 컨테이너 윈도우로 옮기면, 그 창의 부모가 바뀌게 됩니다.
- wnidow의 title이 변경되는 경우: 역시 마찬가지로 젬나이에게 예시를 물어봤다.
> 다이얼로그 제목 변경
> 
> 예: "로그인 오류" 다이얼로그가 처음에는 "Error"라는 제목을 가지고 있다가, 상황에 따라 "Network Error"로 바뀌는 경우.
> 
> 액티비티 창의 제목 변경
> 
> 예: 이메일 앱에서 처음에는 "Inbox"라는 제목을 가진 윈도우가, 사용자가 특정 폴더로 이동하면 "Sent Items"로 바뀌는 경우.
> 
> 팝업 윈도우 제목 변경
> 
> 예: 설정 팝업이 "Settings"에서 "Advanced Settings"로 바뀌는 경우.
- 윈도우가 Pip모드로 변경될 때

**그리고..., 미처 발견하지 못했던 새로운 이벤트 종류들을 발견했다.**

바로, 
- TYPE_GESTURE_DETECTION_END, TYPE_GESTURE_DETECTION_START
- TYPE_TOUCH_EXPLORATION_GESTURE_END, TYPE_TOUCH_EXPLORATION_GESTURE_START
- TYPE_TOUCH_INTERACTION_END, TYPE_TOUCH_INTERACTION_START

이다. 그동안 삼성 ui 등의 런처에서 view가 touch를 감지 하지 못하는 상황에 골치가 아팠다.

아, 근데 사용자의 순수한 '상호작용' 자체에도 이벤트를 발생시킨다니..., 이건 몰랐다.

제미나이의 예시 흐름도를 보겠다.

```
사용자 화면 터치 시작
        │
        ├─ TYPE_TOUCH_INTERACTION_START
        │
        ├─ 손가락을 화면 위에 움직이며 탐색 (TalkBack 등)
        │       ├─ TYPE_TOUCH_EXPLORATION_GESTURE_START
        │       │
        │       └─ 탐색 종료 → TYPE_TOUCH_EXPLORATION_GESTURE_END
        │
        ├─ 시스템이 특정 제스처 인식 시작
        │       ├─ TYPE_GESTURE_DETECTION_START
        │       │
        │       └─ 인식 종료 → TYPE_GESTURE_DETECTION_END
        │
        └─ 손가락을 화면에서 뗌
                └─ TYPE_TOUCH_INTERACTION_END

```

이때, TYPE_TOUCH_EXPLORATION_GESTURE_X는 주로 talkback에 사용되는 기능인데, 다음 영상을 보면 이해가 빠르다.
- [다음 영상](https://www.youtube.com/watch?v=q8THhm0y1GA) 0:53~ 부분을 보면 된다.

그렇다면 약간의 제약만 걸어둔 채로 TYPE_TOUCH_INTERACTION_END를 이용하면, isTouched 플래그가 터치를 했음에도 true로 바뀌지 않는 현상을 해결할 수 있겠다.

아무튼 이렇게 여러가지 AccessbilityEvent를 알아보았다. 이 모든 Event들을 포함하는게 `typeAllMask`이다.

typeAllMask외에도 여러 옵션들이 있었다.

- typeGestureDetectionStart
- typeAssistReadingContext
- typeContextClicked
- typeTouchInteractionEnd
- typeViewClicked

... 가만히 읽어보면 앞에서 살펴봤던 AccessibiltiyEvent들이란 것을 알 수 있겠다.
- [문서](https://developer.android.com/reference/kotlin/android/accessibilityservice/AccessibilityServiceInfo?_gl=1*193qfvs*_up*MQ..*_ga*MTQ1MTE1MDg5MS4xNzc3NzkxODMy*_ga_6HH9YJMN9M*czE3Nzc3OTE4MzEkbzEkZzAkdDE3Nzc3OTE4MzEkajYwJGwwJGg5NzQ2MjQ1NTA.#xml-attributes)만 보아도, xml 버전의 AccessibiltiyEventTypes인 것을 알 수 있다.

이제 세 번째 줄인 `android:accessibilityFlags`로 넘어가겠다.

#### android:accessibilityFlags

[다양한 flag들 공식문서 설명](https://developer.android.com/reference/kotlin/android/accessibilityservice/AccessibilityServiceInfo?_gl=1*1c3iep4*_up*MQ..*_ga*NjkzMDQ2NDIwLjE3Nzc3OTExMTE.*_ga_6HH9YJMN9M*czE3Nzc3OTExMTEkbzEkZzAkdDE3Nzc3OTExMTEkajYwJGwwJGg3Njg5NzIzNTI.#default)

Additional flags as specified in android.accessibilityservice.AccessibilityServiceInfo. 

생각보다 설명이 빈약한 것 같아서 더 찾아봤다. 나오질 않는다. 

android:accessibilityFlags는 접근성 서비스가 어떤 이벤트를 수신하고, 어떤 제스처를 처리하며, 어떤 UI 요소를 탐색할 수 있는지를 시스템에 알려주는 설정이라고 한다. 

각 flag는 아래와 같다.
- 마찬가지로 상수로 flag들이 정의됐다.

*!주의: copilot에게 받아온 답변 그 자체라 검증을 거치지 않았음.*

| Flag | 값 | 설명 | 사용 예시 |
| --- | --- | --- | --- |
| **DEFAULT** | 1 | 기본 상태. 특별한 동작 없음. | 단순 접근성 서비스 기본 설정. |
| **FLAG_ENABLE_ACCESSIBILITY_VOLUME** | 128 | 접근성 오디오 트랙을 별도의 볼륨 스트림(``STREAM_ACCESSIBILITY``)으로 제어. | TalkBack 음성 피드백 볼륨을 따로 조절. |
| **FLAG_INCLUDE_NOT_IMPORTANT_VIEWS** | 2 | 접근성에 중요하지 않다고 표시된 뷰도 포함. | 레이아웃 매니저 같은 뷰까지 탐색 필요할 때. |
| **FLAG_INPUT_METHOD_EDITOR** | 32768 | 접근성 서비스가 일부 IME 기능을 수행할 수 있게 함. | 텍스트 입력 연결, 선택 변경 알림 수신. |
| **FLAG_REPORT_VIEW_IDS** | 16 | AccessibilityNodeInfo에 뷰 ID 포함. | UI 테스트 자동화, 특정 뷰 추적. |
| **FLAG_REQUEST_2_FINGER_PASSTHROUGH** | 8192 | 멀티핑거 제스처 모드에서 두 손가락 제스처를 일반 제스처로 전달. | 두 손가락 스와이프를 단일 손가락처럼 처리. |
| **FLAG_REQUEST_ACCESSIBILITY_BUTTON** | 256 | 내비게이션 영역에 접근성 버튼 표시 요청. | TalkBack에서 접근성 버튼 제공. |
| **FLAG_REQUEST_ENHANCED_WEB_ACCESSIBILITY** | 8 | 웹 접근성 향상 요청 (API 26 이후 Deprecated). | 예전 웹뷰 접근성 개선용. |
| **FLAG_REQUEST_FILTER_KEY_EVENTS** | 32 | 키 이벤트 필터링 요청. | 서비스가 키 입력을 가로채 단축키 처리. |
| **FLAG_REQUEST_FINGERPRINT_GESTURES** | 512 | 지문 센서 제스처 이벤트 수신. | 지문 센서로 제스처 제어. |
| **FLAG_REQUEST_MULTI_FINGER_GESTURES** | 4096 | 멀티 핑거 제스처 활성화 요청. | 두 손가락 스와이프 등. |
| **FLAG_REQUEST_SHORTCUT_WARNING_DIALOG_SPOKEN_FEEDBACK** | 1024 | 접근성 단축키 경고 대화상자에 음성 피드백 요청. | 단축키 활성화 시 사용자에게 음성 안내. |
| **FLAG_REQUEST_TOUCH_EXPLORATION_MODE** | 4 | 터치 탐색 모드 요청. | TalkBack 같은 서비스가 화면 요소를 손가락으로 탐색할 수 있도록. |
| **FLAG_RETRIEVE_INTERACTIVE_WINDOWS** | 64 | 모든 인터랙티브 윈도우 콘텐츠 접근. | 멀티윈도우 환경에서 각 창 탐색. |
| **FLAG_SEND_MOTION_EVENTS** | 16384 | 제스처 탐지 시 MotionEvent도 서비스에 전달. | 제스처 인식 실패/패스스루 이벤트 분석. |
| **FLAG_SERVICE_HANDLES_DOUBLE_TAP** | 2048 | 더블탭 제스처를 프레임워크 대신 서비스가 처리. | TalkBack이 더블탭을 직접 처리. |

지금 우리 앱에서는

```kotlin

android:accessibilityFlags="flagDefault|flagReportViewIds|flagIncludeNotImportantViews|flagRetrieveInteractiveWindows"

```
로 쓰고 있다. 다만, 여기서 enhaned web accessibility가 궁금해서 더 찾아보았다. (구글, 유투브 등 일부 웹 페이지에서, 더보기 창을 눌러 조작할 때 인식을 못하는 현상이 빈번했기 때문이다.)
- 마침 이 현상에 관해 LLM에게 물으니 다음과 같이 말했다. 
> 현재의 관계
> 
> 뷰(View) 이벤트 체계와 웹 콘텐츠는 완전히 동일하지 않습니다.
>
> WebView는 DOM 요소를 AccessibilityNodeInfo로 변환해주지만, 이는 “뷰처럼 보이게 흉내내는 것”이지 실제 뷰 이벤트(TYPE_VIEW_CLICKED)와 1:1 대응은 아닙니다.
>
> 그래서 웹 페이지 버튼을 눌러도 TalkBack이 “버튼 클릭됨”이라고 읽어주지만, 내부적으로는 DOM 이벤트를 Accessibility 이벤트로 변환한 결과일 뿐입니다.

어쩐지 인식을 못 하더라.... 일단 `flagSendMotionEvents`와 `flagRequestEnhancedWebAccessibility`도 추가해주기로 했다.
- API26이후로는 deprecated라는데, 공식 문서에는 딱히 그런 내용이 없어 우선 flagRequestEnhancedWebAccessibility를 넣기로 했다.


#### android:canRetrieveWindowContent

서비스에서 UI 계층 구조를 검사해야 하는 경우 (예: 화면에서 텍스트를 읽기 위해) true여야 합니다. 

그렇다고 한다. 당연히 계층 구조를 몽땅 뽑아와야 하는 우리 앱으로선, true로 설정해야 한다.


#### android:accessibilityFeedbackType

사용자에게 어떤 방식으로 화면 요소를 전달해주느냐, 인데, "feedbackSpoken"가 예시 그대로라 일단 그렇게 했다. 주로 시각 장애인을 위한 talkback기능을 위해서 있는 플래그 같다.


####  android:notificationTimeout

[문서 링크](https://developer.android.com/reference/android/R.styleable?_gl=1*1552zmx*_up*MQ..*_ga*MTc1Njc1NzM4MS4xNzc3ODA0ODYx*_ga_6HH9YJMN9M*czE3Nzc4MDQ4NjAkbzEkZzAkdDE3Nzc4MDQ4NjAkajYwJGwwJGg2NjQ4NTQyNjI.#AccessibilityService_notificationTimeout)에 나와있다.

public static final int AccessibilityService_notificationTimeout

The minimal period in milliseconds between two accessibility events of the same type are sent to this service. 

This setting can be changed at runtime by calling android.accessibilityservice.AccessibilityService.setServiceInfo(android.accessibilityservice.AccessibilityServiceInfo).

May be an integer value, such as "100".

Constant Value: 6 (0x00000006)
> 같은 타입의 접근성 이벤트가 서비스에게 보내질 최소 시간 간격(ms 단위)이다. 공식 문서 예시에서는 100이었으니, 0.1초라고 할 수 있겠다.

이게 매우 중요한 설정값인게, 우리 앱의 경우, TYPE_WINDOW_CONTENT_CHANGED 이벤트가 너무 자주 발생한다. 새로운 창이 열리거나, 화면을 스와이프 할 때마다 우후죽순 동일 타입 이벤트들이 쏟아지는데, 어느정도 필터링하는 효과가 있겠다. 

다만, 예를 들어 사용자가 빠르게 버튼을 왔다갔다 선택하는 식의 조작을 막지는 말아야겠다. 따라서 최대한 같은 타입의 이벤트는 막으면서, 사람의 의도적인 조작은 허용해야 하는데, 현재는 200으로 설정됐다.
- isTouched의 원활한 true 변경을 위해, 사용자의 touch조작을 TYPE_VIEW_CLICKED --> TYPE_TOUCH_INTERACTION_X로 변경했으므로, 0.1초를 추가해준다.
- 300으로 설정.



...writing....