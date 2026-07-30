---
layout: post
title:  "[project] video-streaming 공부"
author: kau-newbie
categories: [project, video, streaming]
image: assets/images/forPost/multimedia.png
---

요약: video streaming이란 무엇인가

# Video Streaming 공부

프로젝트를 진행하다보니, 홈캠으로부터 비디오 스트리밍을 받아올 필요가 있었다.

rasberypi 3 a+라는, 충분하다면 충분하고 모자라다면 모자랄 미니 컴퓨터를 통해 비디오 스트리밍을 쏘려고 하니,

제미나이가 `webRTC`라는 기술을 추천해줬다. 그게 뭔데? 참을 수 없었다. 그래서 공부를 시작하려 한다.

## 제미나이의 로드맵

당장 1도 몰라 막막한데, webRTC는 전문서적이 (인터넷에 쳐보니) 국내서적으로 1권 있었다.

이건, 그냥 인터넷으로 공부하는게 좋을 것 같다.

그래도 로드맵 정도는 있으면 좋겠다 싶어서 제미나이에게 로드맵을 짜달라고 했다. 아래와 같다.

---

#### 미디어 스트리밍 전문가 로드맵

```
[1단계: 스트리밍 기초 & 코덱] ──> [2단계: FFmpeg 마스터] ──> [3단계: WebRTC & NAT 트래버설] ──> [4단계: 파이프라인 통합]
 (프로토콜/코덱 이해)               (영상 처리/변환/송출)             (ICE/STUN/TURN/Signaling)          (MediaMTX/라즈베리파이/Web)

```

##### 1단계: 미디어 스트리밍 기초 & 프로토콜 이해

영상/음성 데이터가 어떻게 압축되고 네트워크를 통해 전달되는지 기본 원리를 파악하는 단계입니다.

* **핵심 학습 주제:**
* **코덱 (Codec):** H.264, H.265 (비디오) / AAC, OpUS (오디오) — 압축 알고리즘 및 I/P/B 프레임 개념
* **컨테이너 (Container):** MP4, MKV, FLV — 비디오+오디오+자막 메타데이터 포맷
* **스트리밍 프로토콜 비교:**
* RTSP (Real-Time Streaming Protocol - CCTV/IP카메라 표준)
* HLS (HTTP Live Streaming - 유튜브/넷플릭스 등 HTTP 기반 스트리밍)
* WebRTC (Web Real-Time Communication - 초저지연 P2P 통신)


##### 2단계: FFmpeg 마스터하기

FFmpeg는 모든 멀티미디어 처리의 "스위스 아미 나이프"입니다. 입출력 스트림 제어와 CLI 명령어를 익히는 단계입니다.

* **핵심 학습 주제:**
* FFmpeg 기본 명령어 구조: `ffmpeg [input 옵션] -i [입력] [output 옵션] [출력]`
* 하드웨어 인코딩/디코딩: 라즈베리파이(`h264_v4l2m2m`), NVIDIA(`nvenc`), Intel(`qsv`)
* 필터 그래프 사용법 (`-vf`): 해상도 변경, 자막 입히기, 오디오 믹싱
* RTSP/RTMP/HLS 프로토콜 변환 및 가상 테스트 스트림 생성 (`testsrc`)


##### 3단계: WebRTC & NAT 트래버설 깊이 파기

웹 브라우저에서 딜레이 없이 실시간으로 영상/음성을 주고받는 WebRTC 심화 단계입니다.

* **핵심 학습 주제:**
* **시그널링 (Signaling) & SDP:** 브라우저와 서버 간 연결 규격(Offer/Answer) 교환
* **ICE Candidate & NAT Traversal (핵심!):** 사설 IP 환경에서 상대방 IP/포트를 찾는 과정
* **STUN & TURN 서버:** P2P 직통로 형성(STUN) 및 방화벽 우회 중계 서버(TURN/Coturn)
* **WHEP (WebRTC HTTP Egress Protocol):** HTTP REST API로 WebRTC 미디어 수신을 주고받는 현대적 표준 프로토콜

##### 4단계: 실전 파이프라인 구축 & 서비스 통합

개념들을 묶어서 라즈베리파이(엣지 디바이스)부터 우분투(미디어 서버), 웹 대시보드까지 완성하는 단계입니다.

* **핵심 학습 주제:**
* 라즈베리파이 카메라 제어 (`Picamera2`) 및 파이썬-FFmpeg 프로세스 연동
* MediaMTX 설정 최적화 (`mediamtx.yml` 커스텀)
* HTML5 `<video>` 태그 및 JavaScript `RTCPeerConnection` / WHEP Client 라이브러리 연동
* 서비스 확장: OpenCV 연동을 통한 실시간 모션 감지 및 AI 객체 인식 추가

#### 학습 진행 팁

1. 처음부터 C++ API를 직접 건드리지 마시고, **FFmpeg CLI 명령어로 영상을 변환하고 송출하는 실습**부터 편하게 시작해 보세요.
2. WebRTC 연결이 안 될 때는 항상 크롬 브라우저의 내부 디버그 탭인 **`chrome://webrtc-internals/`** 페이지를 열어두고 ICE 수신 상태를 눈으로 보면서 공부하시면 훨씬 이해가 잘 됩니다.

---

라고 한다. 당장 시작해보자.

## 1단계: 미디어 스트리밍 기초 & 프로토콜 이해

비디오는 상상이상으로 용량이 크다. 네트워크를 통해 전송할 때나 저장할 때 어마어마한 용량은 곧 돈이므로, **압축**해서 보내고자 했다.
> [안될공학](https://www.youtube.com/watch?v=itdPTcBHRbM)영상을 보면, MEPG이라는 코덱을 사용하면 1/1000까지 압축이 된다고 한다.

### 코덱과 압축 알고리즘

[유투브 영상](https://www.youtube.com/watch?v=AXUZXBYdRc0) <-- 알기 쉽고 좋았던 영상이다.

코덱(**Codec**)은 코더(**co**der)와 디코더(**Dec**oder)에서 따온 말이라고 한다.
- *코더*는 압축을 해준다. 
- *디코더*는 압축을 풀어준다.

이 두 기술들을 합쳐서 *'코덱'*이라고 부른다.

**코덱**은 이 두 기술들을 합쳐 부르기도 하고, 혹은 **압축된 영상을 복원하는 디코더**를 가리키기도 한다.
- 보통 이 두 기술을 합친, `'규칙'`-압축과 복원에 관한-을 말한다.
> 규칙이자 알고리즘이 H. 시리즈나 MPEG 등이 있는 것이다.

또, 이 영상에 따르면, *인코딩*(압축, 또는 변환의 의미를 가지는데, 여기선 '압축')을 거쳐 압축된 영상을 복원하려면,

압축 시 사용한 코더가 필요한데, 이때, 영상 자체를 내가 가지고 있는 코더(엄밀히 말하면 디코더)에 맞는 형식으로 *변환*해줄수도 있다.
- 변환하는 작업도 인코딩(Encoding)이라고 부른다.

이때 복원하는 프로그램이 곧 *디코더*가 되겠다.

그림으로 보면 아래와 같다.

![그림으로 보자면](/assets/images/forPost/videostreaming/videoStreamingExample.png) 

#### 비디오 코덱

비디오 코덱에 관해 보니, 

- **ISO**에서 만든 `MPEG`규격이 있고,
- **ITU**에서 만든 `H. 시리즈`규격이 있다.

둘은 이름만 다른 거고, 혼선을 막기위해 두 단체에서 협력하여 만들고 있는 동일 기술이라고 한다.
(이름이 달라도 같은 기술이면 호환이 된다.)

|ITU-T 표준 이름| MPEG 표준 이름|흔히 부르는 명칭 및 용도|
|---|---|---|
|H.262|MPEG-2 (Part 2)| DVD 및 HD 방송 송출용|
|H.264|MPEG-4 AVC (Part 10)|블루레이, 유튜브, 모바일 스트리밍 (가장 범용적)|
|H.265|MPEG-H HEVC (Part 2)|4K/8K UHD 영상, 고효율 압축 기술H.266MPEG-I VVC (Part 3)VR/AR, 8K 스트리밍용 차세대 코덱|

문제는 내가 프로젝트에 사용할 컴퓨터는 *라즈베리파이 3a+*이고, 해당 기기는 `h.264`를 지원한다고 한다.

[라즈베리파이 webRTC wiki](https://github.com/TzuHuanTai/RaspberryPi-WebRTC/wiki) V4L2 H.264 모드에 관한 내용이 있다.
- v4l2외에도 libcamera를 최신 OS에서 지원&추천 한다는데, uncompressed format만 사용한대서 v4l2 선택지밖에 없다.

위 사이트에서 보면, h.264에 관해 하드웨어 가속을 지원한다고 돼있다.

<details>
  <summary>하드웨어 가속화란?</summary>
    <blockquote>충격적이게도! 하드웨어 가속화란, (인코더/디코더를 예시로 들면) <br>
    특정 소프트웨어를 사용하는게 아니라 특화된 별도의 하드웨어 부품을 쓰는 것이라고 한다.</blockquote>
</details>

(`--hw-accel` 옵션을 읽어보니, *DMA*를 이용한다는 것 같은데, *DMA*란, I/O 연결된 기기가 CPU를 거치지 않고 바로 메모리와 데이터를 주고 받는 방법이다.)

이번에는 더 자세하게, 압축 알고리즘에 대해 알아볼 차레이다.

#### 압축 알고리즘

\<참고>

- [기가 막힌 영상을 하나 찾았다.](https://www.youtube.com/watch?v=MqP8ur-FNuA) 압축에 대해 전체적으로 잘 설명해주는 것 같다.
- [기가 막힌 영상을 또 하나 찾았다.](https://www.youtube.com/watch?v=-4NXxY4maYc) 포맷과 컨테이너에 대해 설명해준다.

아무튼, 영상들을 정리해보겠다.

디지털 비디오 파일들을 보면 extension(확장자)로 `.mov`, `mp4` 등이 붙어있다.

##### Container

이때, 이 확장자들은 **Container**를 나타내는 것이다.
> Container는 Codec과 다르다.
> - video steram과 함께 그 contents를 담고있는 *wrapper*를 나타낸다.

그리고 **비디오 format**은 **Codec** + **Container**를 나타내는 말이다.

*Codec*으로는 위에서 봤던 H. 시리즈가 있고, *Container*로는 `AVI`, `MOV`, `MP4`, `MTS` 등이 있다.

##### Intra Frame, Inter Frame

다시 압축 알고리즘으로 돌아와서, 압축 알고리즘의 두 가지 방식에 대해 정리하겠다.
1. Intra Frame
2. Inter Frame

<details>
  <summary>(참고)</summary>
    <blockquote>디지털 영상은 많은 수의 프레임의 연속 재생이고, 이 하나하나의 프레임들은 010101010.....의 디지털 데이터로 변환한다. encoding이라고 부르는 이 과정에는 압축 알고리즘이 적용된다. 이 변환 과정을 수행하는 SW 혹은 HW를 Codec이라고도 한다.
    </blockquote>
</details>

**1. Intra Frame**

각 프레임 별로 압축을 진행한다. 
- 영상에 나온 가장 쉬운 예시가 `M-JPEG`(Motion-JPEG)이다. 각 프레임을 `JPEG` 형식으로 저장한다.
- 그 외에 `ProRes`, `DNxHD` 등이 있다.

**2. Inter Frame**

중간 중간의 프레임을 *key 프레임*으로 삼고, 나머지 프레임들은 *delta frame*이라는, 변화에 해당하는 정보를 저장한다고 한다.
- 더 작은 용량으로 저장할 수 있게 된다.
- 대표적으로 내가 이번에 프로젝트에 사용할 `H.264`(AVC)가 있다!
> 단, H.264는 `i-frame h.263`라는, intra-frame로 구현된 경우가 있다고도 한다.

##### I-Frame, P-Frame, B-Frame

이런 압축 기술(Compression)에서는 세 타입의 frame이 쓰인다고 한다.

**1. I-Frame**

  i-frame은 자체적으로 모든 픽셀 정보를 포함한다. jpeg로 압축하는 intra-frame 방식이다.

**2. P-Frame**

  p-frame은 이전의 frame으로부터 **예측**한 motion이나 작은 변화에 관한 정보만을 저장한다.

**3. B-Frame**

  b-frame은 이전 & 이후의 양 방향 frame으로부터 **예측**한 변화 정보를 저장한다.
  > 제미나이보고 비유를 들어달랬더니, b-frame을 기준으로 이전 프레임에선 인물을 가져오고, 이후 프레임에선 배경을 가져온댄다.

영상에서 소개하기로는, 모든 압축은

1. 프레임을 블럭 단위로 쪼개고
2. 블럭 별로 변화를 예측해서
3. 잔차(변화 후 픽셀 값 변화)를 계산한 뒤
4. 압축

의 과정들을 블럭마다 반복한다고 말한다.

시간에 따라 전송 매체도 발전하고, 요구사항도 달라지면서(늘어나면서) 매 기술 표준의 발전마다 블럭 사이즈를 더욱 유연하게 정하고, 또 어떤 원리가 들어가고, 이런 식으로 발전해왔다.

이제 비디오 스트리밍 **프로토콜**에 대해 알아보겠다.

### 비디오 스트리밍 프로토콜 비교:

[참고 영상](https://www.youtube.com/watch?v=KX-_fvvmwlA)

**비디오 스트리밍 프로토콜**은, video streaming data의 format과 전송과 그 절차 모두를 정한 약속이다.

프로토콜이 보낸 사람과 받을 사람간 호환이 돼야 영상을 보내고, 또 시청할 수 있다.

#### RTSP (Real-Time Streaming Protocol - CCTV/IP카메라 표준)

RTP(Real-Time Transport Protocol)을 기반으로 동작하며, 받는 쪽에서 Play, Stop, Pause 로 조작할 수 있도록 한다.
- RTP는 UDP를 기반으로하며, 대신 순서를 제어하고, 타임스탬프로 음성&영상 간의 동기화를 맞추며, 전송되는 영상의 코덱 종류를 포함한다.

단, 브라우저에서 재생을 지원하지 않는다고 한다. (특정 종류의 플레이어가 필요하다.)

client 쪽에서 SETUP 신호 후 OK REPLY를 받았을 때, PLAY 신호를 보낸 뒤, OK REPLY 이후 미디어 데이터를 받는다. 

#### RTMP (Real-Time Messaging Protocol)

Adobe에서 만든 TCP기반의 low-latency audio/video 등등 미디어 전송 프로토콜이다.

지금은 없어진 Flash Player로만 브라우저에서도 비디오를 볼 수 있게끔 한다.

TCP 기반이니 핸드셰이크를 거쳐 연결을 유지한다. 

1. Encoder인 비디오 송출 측에서 RTMP를 통해 압축된 영상을 서버로 보내준다.
2. 서버는 중계 역할을 한다.
3. 서버에서 RTMP로 Client에게 바로 보내주거나, CDN을 통해 다른 HLS 등의 프로토콜로 Client들에게 보낸다.
  > 보통 실시간 방송 스트리밍은 다수를 대상으로 보내야하므로 서버가 중계해주는 것일 뿐, Encoder --> Server 간에서 RTMP를 사용한다 생각하면 된다.

#### HLS (HTTP Live Streaming - 유튜브/넷플릭스 등 HTTP 기반 스트리밍)

애플사에서 만든, High quality & reliability 를 보장하는 프로토콜이라고 한다.

특정 포트만을 사용해야하는 RTMP, RTSP와 달리, HTTP 기반이라 **DRM**이란 기술을 사용할 수 있다고 한다.

<details>
  <summary>(DRM이란)</summary>
    <blockquote>
    제미나이한테 물어봤더니 다음과 같다고 한다.<br>
    DRM(디지털 저작권 관리)은 디지털 콘텐츠(동영상, 음악, 전자책, 소프트웨어 등)의 무단 복제, 유출, 불법 공유를 막고 저작권을 보호하는 종합 기술 및 시스템입니다.<br>

    <strong> 핵심 동작 원리 </strong><br><br>

    콘텐츠 암호화: 원본 영상이나 데이터를 강력한 암호 알고리즘(AES 등)으로 패키징하여 암호화합니다.<br>

    라이선스 발급: 암호화된 콘텐츠를 해석(복호화)할 수 있는 '암호화 키(Key)'를 라이선스 서버에 보관합니다.v

    권한 검증: 사용자가 구매/인증을 완료하면, 정해진 조건(재생 기간, 재생 가능 기기 수 등)이 담긴 라이선스와 키를 유저 기기로 전송합니다.<br>

    안전한 재생: 승인된 플레이어(보안 영역)에서만 키를 사용해 영상이 재생되며, 화면 캡처나 녹화를 원천 차단합니다.<br><br>

    <strong>대표적인 DRM 솔루션:</strong><br>

    Google Widevine (안드로이드, 크롬, 스마트 TV 등)<br>

    Apple FairPlay (iOS, Safari, Apple TV 등)<br>

    Microsoft PlayReady (Windows, Edge 등)
    </blockquote>
</details>

그 외에도 **bitrate streaming**을 사용한다고 한다.

인터넷 환경에 따라 *bitrate*를 조절한다는 건데, 실제로 유투브의 경우, 다양한 codec으로 encoding한 뒤 

사용자의 인터넷 환경에 따라 알아서 적절한 *bitrate*, 즉 적절한 codec의 streaming으로 교체해준다고 한다.

마지막으로 **대부분의 modern web browser**에서 지원한다고 한다!

#### DASH

DASH는 ISO에서 정한 기술로, HLS와 마찬가지로 multiple codec을 지원하며, 또 adaptive bitrate streaming을 지원한다.

다만, 구조적 유연성이 DASH 쪽이 더 좋다고 한다. 또, 영상에 따르면 latency도 HLS에 비해 낮았다.

<br><br>
여기까지 보면 내 홈캠 프로젝트에 맞는 프로토콜은 DASH가 아닌가? 싶었다.

RTSP나 RTMP는 브라우저 지원이 안돼고, HLS보다는 DASH가 빠르댄다.
> 물론 앱으로 볼 것을 염두해 둬야 하기에, 아이폰 유저인 나는 HLS를 써야할 것 같다.

#### WebRTC (Web Real-Time Communication - 초저지연 P2P 통신)

[참고]

[인도 분의 webRTC역사와 기술설명](https://www.youtube.com/watch?v=Kn_3uHaKz7Q)
[webRTC와 SPD와 TURN서버까지](https://www.youtube.com/watch?v=bWcNEk0H4Y0)

그동안 real-time communication을 위해 다양한 브라우저에서 다양한 별도 프로그램을 사용했다.

- activeX
- FLash
- JAVA Applet
- 기타

그리고 2011년, 구글에서 오픈한 `WebRTC`는 현대의 대부분 브라우저에서 Built-in(자체내장)돼있다.
















