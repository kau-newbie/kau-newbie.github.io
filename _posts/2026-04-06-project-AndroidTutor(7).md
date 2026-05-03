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

여기서 주목할 것들은 경험상 아래의 세 가지 정도이다.
1. TYPE_WINDOW_STATE_CHANGED
2. TYPE_WINDOW_CONTENT_CHANGED
3. TYPE_WINDOWS_CANGED



typeAllMask외에도 여러 옵션들이 있었다.

- typeGestureDetectionStart
- 




...writing....