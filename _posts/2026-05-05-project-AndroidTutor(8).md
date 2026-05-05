---
layout: post
title:  "[project]AndroidTutor(8) 안드로이드 접근성 서비스에 관한 고찰-2"
author: kau-newbie
categories: [ android, project, accessibilityService, 접근성서비스]
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

아무튼 서비스 내 디바운싱을 하기 위해 젬나이한테 물어보니 `Runnable`과 `Handler`를 알아야 한다고 했다.





