---
layout: post
title:  "[project]Project: AndroidTutor(7) 접근성서비스 필터링 문제"
author: kau-newbie
categories: [ android, os, project]
image: assets/images/prj-androidtutor-basicmodel.jpg
---

# 접근성 이벤트 필터링 로직 문제 (삼성 launcher v.s. android launcher)

지금 앱 개발을 하면서 테스트는 거의 항상 구글 안드로이드 폰으로 하고 있는데, 보통 이벤트 인식문제는 갤럭시 쪽에서 발생한다.

삼성 launcher의 경우, 발생하는 이벤트도, 앱드로어나 시스템 알림 창 등의 구현이 다르기 때문이다.

그래서 그런지 앱드로어나 홈 화면을 넘길 때 삼성 폰에선 isTouched flag가 true로 바뀌지 않는 경우가 종종 생긴다.

나는 다음과 같은 이유를 원인으로 보고 있다.

1. 삼성 ui 런처는 사용자의 터치나 조작을 감지하지만, 이걸 접근성 이벤트로 발생시키지 않는다.
2. 앱 드로어 확인 결과, 삼성 ui 런처에서는 `TYPE_WINDOW_STATE_CHANGED` 가 아닌, `TYPE_WINDOW_CONTENT_CHANGED`가 발생한다.
3. 앱 드로어 자체의 구성 view들이 id가 없다보니, (No Id로 null을 처리하고 있다. (`resourceId?:"No Id"`)) 다른 No Id인 view들의 이벤트에 묻힐 수 있겠다.
- 왜냐하면 현재 다음과 같은 코드로 같은 view id에서 온 이벤트는 일정시간동안 필터링하고 있기 때문이다.
```kotlin
// 특정 resourceID별로 마지막 처리 시간을 저장 (ID : Timestamp)
    // 고정된 크기를 가진 원형 버퍼 형태의 맵
    private val viewEventTimestamps = object : LinkedHashMap<String, Long>(MAX_ENTRIES, 0.75f, true) {
        override fun removeEldestEntry(eldest: MutableMap.MutableEntry<String, Long>?): Boolean {
            return size > MAX_ENTRIES
        }
    }

    val resourceId = event.source?.viewIdResourceName ?: "no_id"
        val currentTime = System.currentTimeMillis()

        val lastTime = viewEventTimestamps[resourceId] ?: 0L
        if ((currentTime - lastTime < VIEW_DEBOUNCE_MILLIS)&&
            (event.eventType == AccessibilityEvent.TYPE_WINDOW_CONTENT_CHANGED)&&
            !node.isCheckable){
            // 너무 빨리 반복해서 발생한 이벤트는 무시 - 단, check 이벤트는 예외.
            return
        }
        // 2. 통과했다면 현재 시간 업데이트
        viewEventTimestamps[resourceId] = currentTime
```
코드에 따르면 TYPE_WINDOW_CONTENT_CHANGED이고, 체크를 사용자가 하고 있는 게 아니라면, view의 id (viewIdResourcName)를 비교해 필터링한다.

여기서 문제는 No Id로 일관하면 다른 view들의 상태변화에 앱 드로어가 열린 이벤트를 필터링해버릴 수도 있는 것이다.

view의 id 만으로는 구분하는게 부족하다. 구분자를 더 넣어야겠다.

1. 당장 떠오르는 건 LLM으로 부터 받아오는 응답의 구분자인 GuideSignature같이 여러 요소를 비교하는 것이다.
2. 이때 이 요소들을 hash 함수에 넣어 그 값들을 비교한다.

hash함수는 이미 버퍼에 넣을 때 hash값을 key 값으로 해서 그 value를 넣고 있기 때문에, 중요한 건 signature를 만드는 작업이다.

이것저것 생각해봤지만, (inteface라던가, data class라던가) 접근성 서비스에서만 쓸 signature이기도 하고, NodeInfo를 받아와야 하기 때문에, 이미 접근성 서비스 안 메서드 getLabelOfNode()를 쓰기 위해 다음과 같이 구현했다.
```
val resourceSignature = resourceId+getLabelOfNode(node)
```
결과는 더 지켜봐야 알겠지만, 당장 문제가 생기는 것 같진 않다.