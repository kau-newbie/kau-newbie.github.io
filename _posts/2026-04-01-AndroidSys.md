---
layout: post
title:  "[Android]view filtering 기능에 대해서"
author: kau-newbie
categories: [ android, os, view, security]
image: assets/images/prj-androidtutor-basicmodel.jpg
---

# Android OS system

## 터치 가로채기(탭재킹) 방지기능

프로젝트 진행중에, 문제가 생겼다. 다음과 같은 상황이 발견됐다.

![이게무슨상황일까](/assets/images/forPost/Androidsys/cantlogin.png)

열심히 검색도 하고, 제미나이에게 물어도 본 결과, 공식 문서에서 그 해답을 찾을 수 있었다.

[view_filtering관련 공식문서](https://developer.android.com/reference/android/view/View#attr_android:filterTouchesWhenObscured)를 보면, 다음과 같이 써있다.

```
Specifies whether to filter touches when the view's window is obscured by another visible window. When set to true, the view will not receive touches whenever a toast, dialog or other window appears above the view's window.
```
그러니까, 지금 상황은 내가 띄운 view(애니메이션)이 로그인 버튼 위에 올라가서 막았다는 말이다. 예를 들면,

![겹치는 영역이 이렇게 있다](/assets/images/forPost/Androidsys/overlayarea.png)

이렇게 그림에서 표시한 부분이 겹치고 있었다.

[view: security관련 페이지](https://developer.android.com/reference/android/view/View?_gl=1*1aur6tx*_up*MQ..*_ga*MTc3NzYzMzE2Mi4xNzc1MDI4NjAw*_ga_6HH9YJMN9M*czE3NzUwMjg2MDAkbzEkZzAkdDE3NzUwMjg2MDAkajYwJGwwJGg4Mjc0Mjk0Njc.#security)에 보면 더 자세히 나와있다.
```
Unfortunately, a malicious application could try to spoof the user into performing these actions, unaware, by concealing the intended purpose of the view. As a remedy, the framework offers a touch filtering mechanism that can be used to improve the security of views that provide access to sensitive functionality.
```
탭재킹에 대해 얘기하고 있다. 원래 버튼을 가리고 그 위에 나쁜 의도의 버튼을 올려 사용자의 터치를 가로챈다고 한다. 내 앱이 malicious application이래..., 슬프다. 이렇게 필터링을 하려면, 
```
logInButton.setFilterTouchesWhenObscured(true);
```
이런식으로 쓰면 된다고 한다. public static final int FLAG_WINDOW_IS_OBSCURED 라는 flag도 함께 세운다.(Constant Value: 1 (0x00000001)) 결과적으로 내 앱에서는 버튼을 가리지 않게 view를 띄우거나, 혹은 login이라는 regex pattern을 view text같은데서 읽으면 선택적으로 멀찍이 애니메이션을 화면상에 띄워야겠다.