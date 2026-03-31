---
layout: post
title:  "[project]Project: AndroidTutor(3) Hilt에 대해서-2"
author: kau-newbie
categories: [ project, android, app, DI, Hilt ]
image: assets/images/prj-androidtutor-basicmodel.jpg
---

# AndroidTutor(가제) 프로젝트(2)

분명 잘 따라가고 있었는데 [Di-Hilt 문서](https://developer.android.com/training/dependency-injection/hilt-android?hl=ko&_gl=1*49kihh*_up*MQ..*_ga*MTU4ODYxOTE0LjE3NzQ4NjUwNDc.*_ga_6HH9YJMN9M*czE3NzQ4NjUwNDYkbzEkZzAkdDE3NzQ4NjUwNDYkajYwJGwwJGg0NDQyMzYzMjU.), 문제가 많았다.

## HILT 한 번 써보려다 고생이 많다. - migration to Ksp

빌드 오류가 너무 많아 애를 먹었다. 우선, 안드로이드 AGP의 버전과 ksp의 버전이 맞지 않는 문제가 끊이질 않았다.
```
Unresolved reference 'ksp'.
``` 
ksp가 여기선 뭔가, 하니 
- KSP (Kotlin Symbol Processing): 코틀린 코드를 분석하는 도구입니다. 버전이 2.0.21-1.0.25 처럼 코틀린 버전과 연동됩니다.
- Hilt (Dagger Hilt): 의존성 주입 라이브러리입니다. 버전이 2.51.1이나 2.3.6 처럼 나갑니다.

라고 한다. 일단 기존에 쓰려던 kapt를 ksp 대신 넣어놓고 계속 오류를 읽어봤다. 자세한 오류 설명을 보니 런타임에서 AGP가 9이상이어야 된댄다.

![버전을 이리저리 바꿔보았지만...](../assets/images/forPost/AndroidTutor(2)/builderrorExample1.png)

대충 요 사진에서 두 번째 줄이 AGP 9.0 이상이어야 된다는 내용이었다. 위 사진은 AGP 바꾸기 싫어서(libs.versions.toml 파일에서 확인해보니, 현재 내 AGP는 8.13.2였다.) ksp를 2.3.6으로 찾아보니 없다는 내용이었다. 물론, 바꾼 근거는 [깃허브 release 페이지](https://github.com/google/ksp/releases)를 보고 바꾼 것이다. 하지만 안됐다. 어렵다.

빌드 오류의 링크 중 하나를 클릭해서, AGP 9.3.1을 다운받았다. 어이쿠, 이번에도 문제가 많다. 

AGP 9.0 부터는 이제 더이상 kotlin을 컴파일하기 위해 외부에서 별도 플러그인을 가져오는 게 아니라, 내장된 코틀린을 써야한댄다. 호다닥 build.gradle.kts (모듈(app)수준부터 프로젝트 수준까지) kotlin 어쩌구 들을 지웠다. 
```
(앱수준 gradle파일에서)
alias(libs.plugins.kotlin.android) 지우고,
kotlinOptions {
            jvmTarget = "11"
        }
도 지우고...,

(프로젝트 수준 gradle파일에서)
alias(libs.plugins.kotlin.android) apply false 도 지우고....
```
하지만, 여전히 빌드는 안됐다. ~~c로 돌아가고 싶다.~~ 해답을 찾기 위해 해당 migration에 관한 공식문서를 찾아봤다.
```
2. 필요한 경우 kotlin-kapt 플러그인 이전
org.jetbrains.kotlin.kapt (또는 kotlin-kapt) 플러그인은 내장 Kotlin과 호환되지 않습니다. kapt를 사용하는 경우 프로젝트를 KSP로 마이그레이션하는 것이 좋습니다.

아직 KSP로 이전할 수 없는 경우 Android Gradle 플러그인과 동일한 버전을 사용하여 kotlin-kapt 플러그인을 com.android.legacy-kapt 플러그인으로 대체합니다.

예를 들어 버전 카탈로그를 사용하는 경우 다음과 같이 버전 카탈로그 TOML 파일을 업데이트합니다.


[plugins]
android-application = { id = "com.android.application", version.ref = "AGP_VERSION" }

# Add the following plugin definition
legacy-kapt = { id = "com.android.legacy-kapt", version.ref = "AGP_VERSION" }

# Remove the following plugin definition
kotlin-kapt = { id = "org.jetbrains.kotlin.kapt", version.ref = "KOTLIN_VERSION" }
```
마침 나도 kapt를 쓰고 있었기에, 게다가 내장 코틀린으로 migration 했기에, 하는 수 없이 ksp로 바꿔야만 했다. 
[migration 관련 문서](https://developer.android.com/build/migrate-to-ksp?hl=ko&_gl=1*1gtk9kk*_up*MQ..*_ga*MTk3OTQ0OTM5OC4xNzc0ODczNzM4*_ga_6HH9YJMN9M*czE3NzQ4NzM3MzgkbzEkZzAkdDE3NzQ4NzM3MzgkajYwJGwwJGgxOTM3ODUyNjE0)
를 참고해서 하나씩 따라가봤다.

일단 AGP_VERSION 저걸 안드로이드스튜디오가 인식을 못했다. 안드로이드 스튜디오에서는 "agp"를 쓰라던데, 바꾸고, 그 외 여러 kotlin 어쩌구 줄을 지우고...
그렇게 되나 싶었는데, 이게 웬걸
```
The project is using an incompatible version (AGP 9.1.0) of the Android Gradle plugin. Latest supported version is AGP 8.13.0
See Android Studio & AGP compatibility options.
```
네? 링크를 타고 확인하러 가봤다.

[이 링크를 타고 갔더니,](https://developer.android.com/studio/releases?hl=ko#android_gradle_plugin_and_android_studio_compatibility)
![26년3월30일기준 안드로이드스튜디오버전](/assets/images/forPost/AndroidTutor(2)/AGP-and-vermatching.png)
아, 나는 옛날 것을 쓰고 있었구나... 또 한세월 걸려 이번에는 안드로이드 스튜디오를 업그레이드 했다. (나는 일각고래 좋아하는데..., 강제로 판다가 됐다.)
그리고 마참내! 됐다! ksp 줄들이 파란걸 볼 수 있다.
![migration to ksp is too hard...](/assets/images/forPost/AndroidTutor(2)/ksp-good.png)

문제는 또 새로운 에러가 떴다는 건데..,
![new err?](/assets/images/forPost/AndroidTutor(2)/newerr1.png)

제미나이에 따르면, 보통 Dagger Hilt 라이브러리와 그 컴파일러역할을 하는 KSP 버전이 서로 일치하지 않기 때문이라고 했다.
Hilt 버전 일치를 먼저 확인하랜다. 다시 gradle.kts 파일을 프로젝트 레벨과 app(모듈) 레벨에서 서로 같은지 확인한다. 아, 서로 버전이 달랐다. 황급히 app 레벨에서 gradle 파일을 수정했다.

![find version of KSP](/assets/images/forPost/AndroidTutor(2)/app-lv-settings.png)

이렇게 하고 돌리니 이제 새로운 에러가 났다.
```
error: [Dagger/MissingBinding] android.content.Context cannot be provided without an @Provides-annotated method.
```
이제부터는 앱 안의 코드 문제이다. 드디어 Hilt를 사용할 준비가 되었다(?) 기쁘다. 

...To Be Countinue...