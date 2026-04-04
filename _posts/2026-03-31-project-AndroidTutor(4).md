---
layout: post
title:  "[project]Project: AndroidTutor(4) Hilt에 대해서-3"
author: kau-newbie
categories: [ project, android, app, DI, Hilt ]
image: assets/images/prj-androidtutor-basicmodel.jpg
---

# AndroidTutor(가제) 프로젝트(4)

[Di-Hilt 문서](https://developer.android.com/training/dependency-injection/hilt-android?hl=ko&_gl=1*49kihh*_up*MQ..*_ga*MTU4ODYxOTE0LjE3NzQ4NjUwNDc.*_ga_6HH9YJMN9M*czE3NzQ4NjUwNDYkbzEkZzAkdDE3NzQ4NjUwNDYkajYwJGwwJGg0NDQyMzYzMjU.)와 유콩씨가 준 자료를 가지고 Hilt를 하나하나 적용해볼 것이다.

## libs.versions.toml 파일에 대해서

사실 그동안은 gradle파일에 버전을 일일히 명시하곤 했었는데, 버전관리를 간편하게 하기 위해 `libs.versions.toml`파일을 사용하는게 대세라고 한다. 이를테면,
```
id("com.google.dagger.hilt.android") version "2.59.2" apply false
```
이렇게, gradle (project level)에 써뒀었다. 아무튼 나도 대세에 따르기로 했다. (흐름을 거슬러선 안돼! 장자의 말이다.)

그럼 libs.versions.toml 파일은 정확히 뭘까? 제미나이가 자세하게도 알려줘서 일단 복붙을 하겠다.
---
안드로이드 개발에서 **버전 카탈로그(Version Catalog)**는 프로젝트에서 사용하는 라이브러리와 플러그인의 버전을 **한 곳(중앙)에서 관리하는 방식**을 말합니다.

예전에는 각 모듈의 `build.gradle` 파일마다 버전을 일일이 적어주었지만, 이제는 `libs.versions.toml`이라는 설정 파일 하나로 통합하여 관리하는 것이 표준이 되었습니다.

#### 버전 카탈로그의 핵심 구성 (`libs.versions.toml`)
프로젝트의 `gradle` 폴더 안에 위치하며, 크게 세 부분으로 나뉩니다.

- [versions]: 라이브러리 버전 번호를 변수로 정의합니다.
- [libraries]: 실제 라이브러리 경로와 위에서 정의한 버전을 연결합니다.
- [plugins]: Gradle 플러그인들을 정의합니다.

##### Step 1: `libs.versions.toml` 수정
```toml
[versions]
# 1. 버전 정의
hilt = "2.51.1" 

[libraries]
# 2. 라이브러리 정의 (alias 설정)
hilt-android = { group = "com.google.dagger", name = "hilt-android", version.ref = "hilt" }
hilt-compiler = { group = "com.google.dagger", name = "hilt-android-compiler", version.ref = "hilt" }

[plugins]
# 3. 플러그인 정의
hilt-gradle = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
```

##### Step 2: `build.gradle`에서 사용
이제 실제 Gradle 파일에서는 긴 버전 번호 대신 설정한 **별명(alias)**을 사용하여 간결하게 불러옵니다.

**[Project 수준 build.gradle]**
```kotlin
plugins {
    alias(libs.plugins.hilt.gradle) apply false
}
```

**[Module(app) 수준 build.gradle]**
```kotlin
plugins {
    alias(libs.plugins.hilt.gradle)
}

dependencies {
    implementation(libs.hilt.android)
    kapt(libs.hilt.compiler)
}
```

##### 3. 버전 카탈로그를 쓰면 무엇이 좋나요?

| 장점 | 설명 |
| :--- | :--- |
| **중앙 집중 관리** | 여러 모듈이 있는 프로젝트에서 한 곳만 수정하면 모든 모듈의 버전이 업데이트됩니다. |
| **자동 완성 지원** | IDE(안드로이드 스튜디오)에서 `libs.` 입력 시 정의한 라이브러리 목록이 자동 완성되어 오타를 방지합니다. |
| **가독성 향상** | `build.gradle` 파일에 지저분하게 나열된 버전 번호가 사라져 코드가 깔끔해집니다. |
| **재사용성** | 설정 파일을 다른 프로젝트로 복사해서 그대로 사용하기 편리합니다. |

> Tip: 안드로이드 스튜디오 최신 버전에서 프로젝트를 새로 생성하면 기본적으로 이 `libs.versions.toml` 방식이 적용되어 있습니다. 직접 문자열로 버전을 적는 것보다 훨씬 안전하고 편리한 방식입니다!

---

이렇게 보면 궁금한 점이 생긴다. "라이브러리는 플러그인과 무슨 차이지?" 굳이 두 개가 따로 있는 이유가 있나? 또 찾아봤다. 내 상황(맥락)에서는,
- 라이브러리: 앱에서 사용할 외부에서 가져온 코드(우리가 잘 아는 그냥 라이브러리)
- 플러그인: 컴파일 시 DI를 위한 코드를 자동으로 생성해주는 도구(tool)

Hilt 특성상, 둘 다 필요하기 때문에, 둘 다 적는다고 한다.

두 번째 질문도 있다. ".toml파일 형식이란?" 아무래도 나한테는 생소했는데, yaml이나 json처럼 구글이 선택한 설정 파일 만들 형식(포맷)이라고 한다.

계속해서 build.gradle.kts파일을 프로젝트 레벨부터 앱 레벨까지 모두 수정해주었다.
```kotlin
//프로젝트 레벨
plugins{
    ...
    alias(libs.plugins.hilt) apply false
    alias(libs.plugins.ksp) apply false
}
//앱(모듈) 레벨
plugins{
    ...
    alias(libs.plugins.hilt)
    alias(libs.plugins.ksp)
}
dependencies{
    ...
    implementation(libs.hilt.android)
    ksp(libs.hilt.compiler)
}
```
다행히 sync를 누르고 나니 별다른 gradle 에러는 없었다. 계속해서, 공식문서에서는 Application을 만들어야 된다고 써있다. 알아보니 Hilt가 DI를 하기 위해선, 해당 앱의 시작점부터 파악하고 있어야 한다. 그리고 그 시작점이 Application 클래스를 상속받는 클래스이다. 별도로 파일을 만들어서 Application을 상속받았다.

![application상속받기](/assets/images/forPost/AndroidTutor(4)/makingapplication.png)




...To Be Continue...