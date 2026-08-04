---
layout: post
title:  "[project] video-streaming 공부"
author: kau-newbie
categories: [project, video, streaming]
image: assets/images/forPost/multimedia.png
---

요약: codec, protocol, picamera2부터 ffmpeg, mediamtx까지

# Video Streaming 공부

홈캠 프로젝트를 진행하다보니, 홈캠으로부터 비디오 스트리밍을 받아올 필요가 있었다.

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
> 
> 미리 스포일러를 하자면, FFMPEG에서는 webRTC를 제공하는 기능이 없다. 따라서 FFMPEG에서 MediaMTX 서버로 보낸 뒤, MediaMTX서버에서 WebRTX로 사용자에게 video streaming을 보낸다. 
> 그리고 ffmpeg에서는 RTSP, RTMP, HLS를 제공한다. 따라서 제일 빠른 RTSP나 RTMP가 이득인 것 같다.

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

WebRTC는 session 방식이다. 즉, 한 번 연결을 만든 뒤, 계속해서 통신을 이어간다. (Stateful)
> UDP를 사용한다.

<details>
  <summary>(udp의 특징들)</summary>
  <pre>강의 자료를 긁어왔다.

  ▪ Process-to-process communication using socket addresses
  ▪ Connectionless Services
    ✓ Independency between datagrams
    ✓ no numbering of datagrams
    ✓ no connection establishment, no connection termination.
    ✓ Message less than 65,507(65535-8(UDP header)-20(IP header)) bytes can use UDP.
  ▪ No flow control, no window mechanism
  ▪ No error control : if the receiver detects an error through the checksum, the datagram is discarded.
  ▪ Checksum
    ✓ A pseudo header, UDP header, application layer data
  ▪ No congestion control
  ▪ UDP is a connectionless simple protocol with optional checksum.
  </pre>
</details>

webRTC 설명을 보면 UDP, SDP 등 프로토콜이 자주 등장하는데, 여기서 잠깐 짚고 넘어가자면, WebRTC는 단일 응용 계층 프로토콜이 아니다.

다음과 같이 여러 계층 프로토콜을 묶은 기술 스택이다.

| OSI 계층|프로토콜/ 기술|역할 |
|---|---|---|
|7. 응용 계층 (Application)|WebRTC (JavaScript API)|"미디어 캡처, PeerConnection 제어, DataChannel 제어"|
|6. 표현 계층 (Presentation)|"Codecs (Opus, VP8/VP9, H.264)"|오디오/비디오 압축 및 복호화|
|5. 세션 계층 (Session)|SDP / ICE / STUN / TURN,"미디어/네트워크 정보 협상| P2P NAT 통과 세션 수립"|
|4. 전송 계층 (Transport)|DTLS / SRTP / SCTP(기기 간 기반: UDP)|SRTP: 미디어 데이터(음성/영상) 암호화 전송SCTP: 데이터 채널(텍스트/파일) 전송DTLS: 키 교환 및 보안 터널 형성|
|3. 네트워크 계층 (Network)|IP (IPv4 / IPv6)|패킷 주소 지정 및 라우팅|

예를 들어보자. 두 명의 user가 연결을 맺으려고 시도한다. 이걸 **signaling**이라고 한다.
> 이 signaling은 '서버'가 주선해주어야 가능하다.

서버에서는 두 user간 signaling을 위해 `SDP`나 `ICE Candidate`를 주고 받는 것을 돕는다.

  - `SDP`란 무엇일까? 
    > SDP는 *Session Description Protocol*의 약자로, 서로 통신시 사용할 설정을 주고받는 방식이다.
    > - 미디어 유형, codec, 화면해상도, ice 등의 정보가 들어있다.

  - `ICE`란 무엇일까? 
    > *Interactive Connectivity Establishment*의 약자로, p2p 연결시 여러 가능한 루트 candidate 중에서 어느 루트로 연결할 것인지 정하는 알고리즘이다.
    > - **Host candidate**: 같은 로컬 네트워크에 속해있는 경우. 같은 공유기를 타고 나가기 때문에 *사설 ip*와 *포트 번호*만으로도 쉽게 통신 가능하다.
    > - **Server Reflexive candidate**: 서로 다른 네트워크에 속해있어서 상대방의 IP 주소를 알아낼 **중계 서버(STUN 서버)**가 필요하다.
    > - **Relay Candidate**: 네트워크 환경에 따라 (방화벽 등등의 요인으로) STUN 서버가 안될 때, **TURN 서버**를 두고, 아예 스트리밍 자체를 중계로 하는 경우이다. 그만큼 delay가 생긴다고 한다.

    <details>
      <summary>(hls의 상대적으로 높은 latency는 turn 서버의 latency와 같은 이유인가? 싶었다.)</summary>
      HLS의 지연은 "데이터를 만드는 방식(애플리케이션/프로토콜 구조)" 때문이고, TURN 서버의 지연은 "데이터가 이동하는 경로(네트워크 라우팅/패킷 처리)" 때문이다.
    </details>

예를 들어, 대부분의 user들은 "집 안 공유기 --> 모뎀 --> 아파트 공용 아이피 --> 외부 인터넷" 의 루트로 인터넷을 사용할텐데, 이때 꼭 **public ip**가 필요하고, 이를 서버에서 알아봐주기 때문이다. 
- NAT(사설 ip와 공인 ip를 변환해주는 기술)에는 여러 보안 기술이 적용되는데, 그 중 하나가 "일단 내가 상대방에게 패킷을 보내야지만" 그쪽으로부터 오는 패킷을 받을 수 있게끔 해놓은 기술이다.

즉, 중앙에서 signaling을 중계할 서버가 필요한 이유는, 연결이 설정되기 전까지는 SDP나 ICE candidate을 서버가 중계해서 전달해줘야 하기 때문이다.
> 이때! SDP교환 이후 서버를 통해 알아내는 것은 본인의 공인 IP임으로, 서버가 중계해서 서로에게 상대방의 IP와 포트 번호를 알려준다.
> - 이게 바로 ICE Candidate를 주고 받은 것이다.

제미나이의 요약 흐름도는 아래와 같다.

```

[1단계] SDP Offer / Answer 교환 (시그널링 서버)
   │  "미디어 종류는? 무슨 코덱 쓸까? 해상도는? 오디오는 포함해?" (미디어 스펙 합의)
   ▼
[2단계] STUN 서버 조회
   │  "내 공인 IP랑 포트 번호가 몇 번이지?" (각자 자기 주소 확인)
   ▼
[3단계] ICE Candidate 교환 (시그널링 서버)
   │  "나 211.x.x.x:10000 으로 문 열 수 있어!" (서로의 주소 명함 교환)
   ▼
[4단계] P2P 직통로 연결 완료 (UDP Hole Punching)
      "문 다 열렸으니 영상/음성 스트림 직접 쏜다!" (P2P 미디어 송수신)

```

이때! 서버와 user간의 프로토콜은 `http`, `webSocket` 등을 사용한다.

그 외에도 UDP를 위한 security protocol인 `DTLS` (패킷 암호화를 진행한다.), 그리고 연결간 참가자 id 식별이나 QoS 상태, 그리고 이 상태에 따라 bitrate를 조정하는 `RTCP`가 있다.

##### WebRTC in JavaScript

추가로, **JavaScript에서 API 형태로 webRTC를 지원한다.**

## 각 라이브러리는 대체 여기서 무엇을 하지?{:#question1-section}

이때, architecture를 보면, 궁금한 점이 생긴다.

제미나이가 추천해준 mediaMTX, FFmpeg, rasberypi의 카메라 라이브러리 등은 각각 어떤 용도인가? 하는 것인데, 

우선, rasberypi 3a+에서 쓰고 있는 `picamera2`가 할 수 있는 일/ 없는 일부터 나눠야 하겠다.

**picamera2**란, [pdf](https://pip-assets.raspberrypi.com/categories/652-raspberry-pi-camera-module-2/documents/RP-008156-DS-3-picamera2-manual.pdf)에 따르면,

USB 포트로 연결하는 rasberypi 보다는, 길다란 납작 케이블 형태의 카메라를 python으로 조작하기 위해 만들어진 라이브러리라고 한다.
> 문서에 따르면 아주아주 고수준의 api들도 제공한다고 한다.
>
> 앞서 살펴봤던 libcamera나 vl42 기반의 고수준 라이브러리이다.

Video 파트 문서를 읽어보자.

### Encoder와 start_recording, start_encoding

```py
from picamera2.encoders import H264Encoder
from picamera2 import Picamera2
import time
picam2 = Picamera2()
video_config = picam2.create_video_configuration()
picam2.configure(video_config)
encoder = H264Encoder(bitrate=10000000)
"""혹은,
  encoder = H264Encoder()
  picam2.start_recording(encoder, 'test.h264', quality=Quality.HIGH)
"""
output = "test.h264"
picam2.start_recording(encoder, output)
time.sleep(10)
picam2.stop_recording()

```
코드가 몹시 예쁘다. 단순히 Picamera2()라는 객체에서 `encoder`와 `ouput 종류`를 정해주고 `start_recording`이라는 편리한 메서드를 사용해 시작할 수 있다.
> 원칙적으로 camer에서 나오는 every frame은 encoder로 간다고 한다. 단, `encoder.frame_skip_count = `같은 설정으로 몇 개의 frame을 건너띄게 설정할 수도 있다고 한다.

<details>
  <summary>프로젝트에 적절한 인코더인 <strong>h264encoder</strong>도 찾을 수 있었다.</summary>
  <pre>
    7.1.1. H264Encoder

    The H264Encoder class implements an H.264 encoder using the Pi’s in-built hardware, accessed through the V4L2 kernel drivers, 
    supporting up to 1080p30. The constructor accepts the following optional parameters:

    • bitrate (default None) - the bitrate (in bits per second) to use. The default value 
      appropriate bitrate according to the Quality when it starts.

    • repeat (default None will cause the encoder to choose an False) - whether to repeat the stream’s sequence headers with every    Intra frame (I-frame). 
      This can be sometimes be useful when streaming video over a network, when the client may not receive the start of the stream where the sequence headers would normally be located.

    • iperiod (default None) - the number of frames from one I-frame to the next. 
      The value None leaves this at the discretion of the hardware, which defaults to 60 frames.
    This encoder can accept either 3-channel RGB...
  </pre>
</details>

### Output

출력 방식에 대한 내용도 찾을 수 있었다. 이 부분이 상당히 중요했다!

```
Output objects: 인코더가 만든 인코딩된 비디오 프레임을 직접 받아서 파일이나 네트워크 소켓으로 전달하는 역할을 합니다.

생성 방식:

  보통은 생성자(Constructor)를 이용해 직접 Output 객체를 만듭니다.

  하지만 start_encoder()나 start_recording() 같은 함수에 단순 문자열(파일 이름 등)을 넘기면, 내부적으로 자동으로 FileOutput 객체가 만들어져서 파일에 기록됩니다.

```

다양한 output fotamt이 있었다.

1. FileOutput
  - 데이터를 파일로 저장.
  - 예: 사진을 .jpg, 영상은 .mp4 같은 파일로 저장.
  - 간단히 결과물을 남기고 싶을 때 사용.

2. CircularOutput
  - 데이터를 메모리에 원형 버퍼 형태로 저장.
  - 최근 몇 초간의 영상만 유지 → “타임머신”처럼 직전 순간을 저장 가능.
  - 보안 카메라나 이벤트 감지용으로 유용.

3. NullOutput
  - 데이터를 버려버리는 출력.
  - 실제 저장은 안 하고, 단순히 카메라 파이프라인을 유지할 때 사용.

4. FfmpegOutput
  - 데이터를 FFmpeg 프로세스에 전달.
  - FFmpeg은 강력한 영상 처리 도구로, 인코딩·스트리밍·변환을 담당.
  - Picamera2에서 찍은 영상을 실시간으로 서버로 스트리밍하거나, 다른 포맷으로 변환할 때 사용.

사실 여기에 넣진 않았지만, `PyavOutput`라는 포맷도 있다. 설명을 보면,
  > The PyavOutput is a more recent integration of Picamera2 with FFmpeg. Rather than passing frames to an external FFmpeg process, 
  > we pass them directly to the FFmpeg libraries running in the same Python process using the PyAV Python bindings.

라고 하는데, `PyAV`에 대한 기반지식이 필요하다고 한다....

이 `PyavOutput`와 `FfmpegOutput` 각각 내부적/ 외부적으로 ffmpeg을 호출해 streaming하는 기능을 제공한다. 일종의 파이프라인이다!
> 내가 직접 ffmpeg 프로그램을 짤 필요가 없다. 해당 output으로 설정해두면, 알아서 ffmpeg 호출 후 메모리에 올려 실행까지 해주는 아주 편리한 기능이었다! 감사합니다...

다만, **우리 코파일럿씨의 의견으로는,** `FFMPEG`**이 훨씬 프로젝트에 적합하다고 한다.**

다음과 같은 이유가 있다.

1. 편의성: FFmpeg은 패킷 손실 복구, 버퍼링, 재연결 같은 기능을 이미 구현해두었다. 따라서 `PyAV`는 직접 코드를 짜야한다고 한다.
2. 프로토콜 제공폭: `PyAV`는 RTMP, RTSP, HLS, SRT 등 스트리밍 표준 프로토콜을 지원하지 않았다. -> 단순 소켓통신을 위한 tcp, udp는 제공하나, 핸드셰이킹부터 이것저것 구현해야한다. 반면, `FFMPEG`은 지원한다!
3. 하드웨어 가속: 하드웨어 가속 측면에서도 `PyAV`는 python 레벨에서 다루기 때문에 제한적이라고 한다.

그럼 FFMPEG output으로 가야겠다. 예제 코드를 살펴본다.

```py

from picamera2.outputs import FfmpegOutput
output = FfmpegOutput("test.mp4", audio=True)

```

이런식으로 FfmpegOutput을 쓰면서, 파일로 저장하고, audio도 마찬가지로 인코딩해서 저장한다.

우리는 `FFmpeg`을 외부에서 호출한 뒤 이용하는 `FfmpegOutput`을 최대로 활용하기 위해서, 외부의 mediaMTX 서버로 Ffmpeg을 통해 보낼 것이다.
- 코파일럿씨 말을 들어보니, **FfmpegOutput() 자체가 Ffmpeg에게 실행할 명령어 옵션들을 넘겨주는 것이다.**
- 다시한번 상기하자면, 나는 `rtsp`를 이용할 것이다.

예제 코드는 아래와 같다.

```py

output = FfmpegOutput("rtsp://mediamtx-server:8554/mystream")

```

+ 생각보다 별 도움이 안됐던(?) 참고: [picamera2 예제들](https://github.com/raspberrypi/picamera2/blob/main/examples/capture_stream_udp.py)

#### 아키텍처 {#architecture-section}

지금까지 나온 내용들로 홈캠 프로젝트에 적용해보자면, 다음과 같은 아키텍처를 그릴 수 있을 것 같다.

![홈캠프로젝트](/assets/images/forPost/videostreaming/prj-homecam-architecture.drawio.png)

##### pi 에서는,
- cam -> picamera2 -> Ffmpeg 을 거치며 최종적으로 rtsp 프로토콜을 통해 mediaMTX 서버로 스트리밍을 보낸다.

##### mediaMTX에서는, 
- 스트리밍을 받아서 webRTC로 변환한 뒤, client들에게 뿌릴 것이다.
- 물론, signaling 서버를 겸한다. (mediaMTX에 내장돼있다고 한다.)


## 2단계: FFmpeg 마스터하기

webRTC를 이용할 아키텍처는 완성됐고, 그럼 이제 FFmpeg은 어떤 것이고, 어떻게 적용될까?

### FFmpeg이란?

decode, encode, transcode, mux, demux, stream, filter and play 를 지원하는 프레임워크라고 한다.

[공식문서](https://ffmpeg.org/ffmpeg.html#Synopsis)를 읽어보면,

```

ffmpeg [global_options] {[input_file_options] -i input_url} ... {[output_file_options] output_url} ...

```

라는 기본 명령어를 기준으로 설명하고 있다. 글로벌 옵션이란 모든 인풋/아웃풋에 적용되는 옵션이고, 그 외 옵션은 그 다음 나오는 파일에만 적용된다고 한다.

```

입력은 -i 옵션으로 지정하고, 출력은 그냥 URL이나 파일명으로 지정합니다.

명령줄에서 옵션으로 해석되지 않는 문자열은 모두 출력 URL로 간주됩니다.

```
라는데, 다음과 같은 예시를 두고 있다.

Set the video bitrate of the output file to 64 kbit/s:
> ffmpeg -i input.avi -b:v 64k -bufsize 64k output.mp4
> - `-b`로 bitrate를 조절하겠다는건데, `:v`로 비디오 스트림에만 적용하겠다는 뜻이다. 오디오면 `:a`이다.

Force the frame rate of the output file to 24 fps:
> ffmpeg -i input.avi -r 24 output.mp4

`-i` 옵션 뒤에는 input 파일이 나오고,  `-b`로 bitrate를 설정하거나, `-r`로 frame rate를 설정한다.
- input으로 넘어오는 스트림에 대해 30fps로 강제하는 상황도 생각해볼 수 있겠다.(물론 통신환경을 고려해서, 적용하진 않을 것이다.)

`-codec` (또는 `-c`) : copy would copy all the streams without reencoding. 이렇게 `-codec` 옵션 등 정말 다양한 옵션이 많다.
> -c 옵션은 encoder이자 decoder일 수 있는데, 대부분 encoder로 해석하고, 보통 디코더는 `-dec`로 명시해주는 것 같았다.
> 예를 또 들어보자면, 비디오에 대해서는 x264 ([출처](https://ffmpeg.org/ffmpeg-codecs.html#libx264_002c-libx264rgb))일 때, `-c:v libx264`와 같은 식이었다.

그리고 [ffmpeg protocol문서](https://ffmpeg.org/ffmpeg-protocols.html#rtsp)를 보면,

```

rtsp://hostname[:port]/path

```

과 같이 `input`, 혹은 `output쪽 url`에 쓰도록 되어있다.

다음과 같은 방식으로 다양한 옵션들도 쓸 수 있었다.

```py

# Watch a stream over UDP, with a max reordering delay of 0.5 seconds:

ffplay -max_delay 500000 -rtsp_transport udp rtsp://server/video.mp4

```

`-disable` 옵션을 통하지 않는 이상, 모든 프로토콜을 받을 수 있게 default 값이 설정됐으므로, input은 그대로 둔다. ouput에만 `rtsp://hostname[:port]/path`와 같이 적어두면 되겠다.
> 우리는 ffmpeg을 picamera2에서 호출하기 때문이다.

**그리고 무엇보다 WebRTC는 지원하지 않았다. transcoding 해줄 서버가 필요한 근거가 됐다.**

## 3단계: WebRTC & NAT 트래버설 깊이 파기

핵심 학습 주제:
- [x] 시그널링 (Signaling) & SDP: 브라우저와 서버 간 연결 규격(Offer/Answer) 교환
- [x] ICE Candidate & NAT Traversal (핵심!): 사설 IP 환경에서 상대방 IP/포트를 찾는 과정
- [x] STUN & TURN 서버: P2P 직통로 형성(STUN) 및 방화벽 우회 중계 서버(TURN/Coturn)
- [ ] WHEP (WebRTC HTTP Egress Protocol): HTTP REST API로 WebRTC 미디어 수신을 주고받는 현대적 표준 프로토콜

WHEP을 제외하곤 위에서 개념들은 살펴봤으니, WHEP을 배운 후 빠르게 구현단계로 넘어가자.

### WHEP이란

[http.dev](https://http.dev/whep#usage)에 따르면,

WebRTC는 ICE, DTLS, SRTP 같은 transport를 지원하지만, signaling 부분에 대해선 정해놓지 않았었다.

때문에 개별 회사들마다 별도의 websocket이나 기타 방법으로 SDP 교환 방식을 구축해왔다.

결국 서로 다른 회사들마다 호환 가능한 표준이 필요했고, 그래서 나온 게(서버-> 클라이언트 방향이다.) `WHEP`이다.
> encoder --> server는 `WHIP`이라고 한다.

- WHIP이나 WHEP이나 동일한 인증 토큰, ice를 위한 헤더 형식, SDP offer via HTTP POST, receives an SDP answer in the Response 등등 동일하다.
- 따라서 개발자 입장에서는 API를 통해 하나의 로직을 재사용할 수도 있다고 한다.

아무튼 간단하게만 알아보고, 얼른 구현 단계로 넘어가야겠다.

## 4. 구현단계

### MediaMTX

mediaMTX는 `rtsp`, `rtmp`, `srt`, 혹은 `WHIP`으로 들어온 스트리밍을 `WHEP`으로 바꿔줄 **하나의 완성형 APP**이다.
> Go lang으로 만들어졌다고 한다. 
> - "ready-to-use, zero-dependencies"라고 한다. 

설정파일 `mediamtx.yaml`만 수정해서 실행하면 별도의 서버 프로그램으로 동작한다고 한다.

이쯤되니 걱정되는게, 지금 웹 서버에서도 대시보드를 통해 홈캠 영상을 보거나, 혹은 추후 앱으로 확장하려 하는데,

`webRTC`는 **p2p**가 아닌가? 찾아보니 다행히 mediaMTX를 통해 1:N이 가능하다고 한다.

아키텍처는 아래와 같겠다. client와 was 서버로 1:N webRTC 스트리밍을 하는 mediaMTX 서버가 있다.

![아키텍처](/assets/images/forPost/videostreaming/architecturev2.png)

아무튼 계속해서 mediamtx 사용법을 공부해보겠다.

[공식문서](https://github.com/bluenviron/mediamtx)에 따르면, 

**mediamtx**는 live media server and media proxy으로,

- Publish live streams to the server with Media-over-QUIC, SRT, WebRTC, RTSP, RTMP, HLS, MPEG-TS, RTP, using FFmpeg ... *web browsers* and more.
- Read live streams from the server with Media-over-QUIC, SRT, WebRTC, RTSP, RTMP, HLS, using FFmpeg, ... web browsers and more.

라고 한다. `publish`로 mediamtx에게 stream을 보낸 뒤, client들은 `read`로 mediamtx로부터 stream을 받는다.

#### Docker Image Download

```

docker run -it(혹은 -d나 -dit 등등) \
-e MTX_RTSPTRANSPORTS=tcp \
-e MTX_WEBRTCADDITIONALHOSTS=192.168.x.x \    
-p 8554:8554 \
-p 1935:1935 \
-p 8888:8888 \
-p 8889:8889 \
-p 8892:8892 \
-p 8890:8890/udp \
-p 8189:8189/udp \
-p 8892:8892/udp \
bluenviron/mediamtx:1

```
`MTX_RTSPTRANSPORTS=tcp`는 클라이언트가 RSTP프로토콜을 이용해 mediaMTX로 publish 할 때, tcp로 전송하도록 강제하는 옵션이다.
> 도커를 쓰면 ip나 포트가 변경될 수 있어서 udp는 연결이 끊길 수도 있다.
> 
> 혹은, udp를 그대로 쓰고 싶다면, `--network=host`옵션을 써서 도커 컨테이너가 호스트 머신의 network를 그대로(ip 주소나 포트번호 등) 쓴다.
> - 단, 리눅스환경에서만 가능하다.

`MTX_WEBRTCADDITIONALHOSTS`는 클라이언트가 접속할 수 있는, 혹은 상대방이 접속할 수 있는 추가적인 public ip 주소이다.
> 예를 들어, mediaMTX서버가 NAT 뒷 단에 위치해 있을 경우, SDP를 교환해야하는 mediaMTX서버와 클라이언트는 서로 SDP를 주고 받을 수 없다.(주소를 모르기 때문이다!)
>
> 보통은 서버를 두고 해결하지만, 여기서는 이 `MTX_WEBRTCADDITIONALHOSTS`에 적어준 public ip 주소를 통해 클라이언트가 접속할 수 있게끔 하는 추가적인 public ip주소인 것이다.


#### mediaMTX architecture

아래는 mediaMTX architecture 페이지를 그대로 직역했다.

##### MediaMTX의 네트워크 동작

- 외부 소스와 연결 (클라이언트 역할)  
> 설정 파일에 정의된 외부 소스로부터 스트림을 가져온다. 즉, MediaMTX가 클라이언트처럼 동작해 외부에서 영상을 pull 해온다.

- 서버 역할  
> RTSP, RTMP, WebRTC, SRT, HLS 같은 여러 프로토콜을 지원하는 서버를 열어, 클라이언트가 스트림을 publish(송출)하거나 read(시청)할 수 있게 한다.

- 재생 서버  
> 디스크에 저장된 스트림을 읽어올 수 있는 playback 서버를 제공한다.

- 관리 서비스  
> metrics, pprof, Control API 같은 관리용 서비스도 노출한다.

##### 내부 구성 요소

- Path Manager  
> 경로(path)를 관리하고, 인증을 수행하며, 클라이언트를 해당 경로에 연결해준다.

- Paths  
> 각 path는 하나의 스트림을 담고 있으며, 이는 단일 publisher(송출자) 또는 단일 외부 소스가 제공하고, 여러 reader(시청자)에게 방송된다.

- Recorder  
> 스트림을 디스크에 저장하는 역할을 한다. --> 현재 프로젝트에선 필요 없는 기능.

- 구성 관리  
> 모든 동작은 설정 파일이나 환경 변수로 제어된다.

- 인증 기능
> 필요하다면 MediaMTX 서버를 identity 서버와 연동해서 인증을 수행할 수 있다. --> 상당히 중요한 부분일 것이다.

#### publish & read

[공식문서](https://mediamtx.org/docs/features/basic-usage)에 따르면,

간단히 mediaMTX서버를 향해 (위에서 ffmpegOutput()에 적었듯)

```

ffmpeg rtsp://localhost:8554/mystream

```

만으로 publish를 한 것이라고 한다. 읽을 때(read)도 어떤 프로그램을 이용하든 마찬가지로 읽을 수 있다.

예를 들면, `ffmpeg`을 이용해서,

```

ffmpeg -i rtsp://localhost:8554/mystream -c copy output.mp4

```

와 같이 input(`-i`)을 mediaMTX server로 설정해서 받을 수 있다.

##### publish 파트

더 자세히 들어가보자. 먼저, **publish** 부분이다.

[publish](https://mediamtx.org/docs/features/publish)문서는 재밌는 게 있었다.

Live streams can be published to the server with the following protocols:

```

Media-over-QUIC clients
SRT clients
SRT cameras and servers
WebRTC clients
WebRTC servers
RTSP clients
RTSP cameras and servers
RTMP clients
RTMP cameras and servers
HLS cameras and servers
MPEG-TS
RTP

```
라는데, 여길 보면 *rtsp client*도 있고, *cameras and servers*도 있었다.

차이점은 `client`는 직접 mediaMTX server로 `push(publish)` 하는 것이고, `cameras and servers`는 통로만 열어두고 mediaMTX 쪽에서 `pull`해가는 것이다.

client 방식은 위에서 한 ffmpeg rtsp://100.x.x.x 이고, server 방식은 아래와 같다. mediaMTX의 `mediamtx.yaml`파일에서 아래와 같이 작성한다.

```yaml

paths:
  proxied:
    # url of the source stream, in the format rtsp://user:pass@host:port/path
    source: rtsp://original-url


```

여기서 또 선택의 기로가 생겼는데, 막상 듣기로는 mediaMTX로 '쏴주는' 게 훨씬 보안 측면에서 유리할 것 같았다.

이건 코파일럿씨에게 한 번 비교해달라고 부탁했다.

```

🔐 Push 방식 (Client → Server)
- 장점
  - 서버 쪽에서 외부 카메라에 접속할 필요가 없음 → 외부 네트워크에 대한 접근 권한을 최소화.
  - 서버는 단순히 “받는 입장”이라서 공격 표면이 줄어듦.
  - 클라이언트가 인증을 거쳐 송출하도록 설계하면, 서버는 인증된 소스만 받음 → 접근 제어가 명확.
  - NAT, 방화벽 환경에서도 클라이언트가 outbound 연결만 하면 되므로 보안 정책과 잘 맞음.
- 단점
  - 클라이언트 장치(라즈베리파이)가 직접 서버에 연결해야 하므로, 송출 장치가 노출될 수 있음.
  - 네트워크가 불안정하면 송출이 끊길 수 있음.

---

🔐 Pull 방식 (Server → External Source)
- 장점
  - 서버가 직접 외부 카메라를 관리하므로 중앙집중적 제어가 쉬움.
  - 클라이언트 장치가 단순히 RTSP 서버만 열어두면 되므로 설정이 단순할 수 있음.
- 단점
  - 서버가 외부 네트워크로 접속해야 하므로, 방화벽/보안 정책에서 inbound rule을 열어야 함 → 공격 표면 확대.
  - 외부 카메라가 인터넷에 노출되어야 하므로, 카메라 자체가 공격 대상이 될 수 있음.
  - 인증을 제대로 구성하지 않으면 서버가 아무나 접근 가능한 카메라를 가져오게 될 위험.

---

📌 결론
- 보안 관점에서는 Push 방식이 더 유리합니다.  
  - 서버는 외부에 불필요하게 접속할 필요가 없고, 단순히 인증된 클라이언트가 보내주는 스트림만 받으면 되니까요.  
  - 특히 라즈베리파이 같은 장치가 내부망에 있고, MediaMTX 서버가 중앙에 있다면 Push 구조가 더 안전하고 관리하기 쉽습니다.  

---

즉, Picamera2를 쓰고 계신 상황이라면 라즈베리파이가 FFmpeg 클라이언트로 MediaMTX에 push하는 구조가 보안적으로도 더 낫습니다.  

```

push 방식으로 가도록 하겠다. 계속해서 `read` 파트로 넘어가본다.

##### read 파트

```html

<iframe src="http://mediamtx-ip:8889/mystream" scrolling="no"></iframe>

```
와 같이 손쉽게 html의 \<iframe> 태그로 띄울 수 있었다.

그 외에 javascript를 통해 token이나 유저명 등 인증체계를 거칠 수 있는 기능들이 있었는데, 이럼 차피 개발자 도구에서 다 보이지 않나? 하고 생각이 들었다.
> 보안 문제는 추후에 지속적인 공부와 함께 수정해나가야할 것이다.

#### Configuration

이제 configuration이다. 

예시로 `webRTC` 설정을 들어보겠다.

```yaml
###############################################
# Global settings -> WebRTC server

# Enable the WebRTC server, which allows to publish and read streams with the WebRTC protocol.
webrtc: true
# Address of the WebRTC TCP/HTTP listener.
webrtcAddress: :8889  # 기본적으로 webRTC는 8889 port로 열어두고 (url창에서 :8889를 붙여주면되겠다.)
# Enable HTTPS.
# This covers only the WebRTC handshake, while WebRTC streams
# are always encrypted with a key that is exchanged during the WebRTC handshake.
webrtcEncryption: true
# Path to the server key.
# This can be generated with:
# openssl genrsa -out server.key 2048
# openssl req -new -x509 -sha256 -key server.key -out server.crt -days 3650
webrtcServerKey: /저장경로/server.key                # 받아온 key와 certificate를 경로로 적어준다!
# Path to the server certificate.
webrtcServerCert: /저장경로/server.crt
# Allowed CORS origins.
# Supports wildcards: ['http://*.example.com']
webrtcAllowOrigins: ["*"]
# IPs or CIDRs of proxies placed before the WebRTC server.
# If the server receives a request from one of these entries, IP in logs
# will be taken from the X-Forwarded-For header.
webrtcTrustedProxies: []
# Address of a UDP/ICE listener that will receive connections.
# Use a blank string to disable.
webrtcLocalUDPAddress: :8189                                  # ice를 udp로 주고받는 경우.
# Address of a TCP/ICE listener that will receive connections.
# This is disabled by default since TCP is less efficient than UDP and
# introduces a progressive delay when network is congested.
webrtcLocalTCPAddress: ""                                   
# WebRTC clients need to know the IP of the server.
# Gather IPs from interfaces and send them to clients.
webrtcIPsFromInterfaces: true
# Interfaces whose IPs will be sent to clients.
# An empty value means to use all available interfaces.

# Interfaces whose IPs will be sent to clients.
# An empty value means to use all available interfaces.
webrtcIPsFromInterfacesList: []
# Additional hosts or IPs to send to clients.
webrtcAdditionalHosts: ["myserver-dns"]                                 
# ICE servers. Needed only when local listeners can't be reached by clients.
# STUN servers allow to obtain and share the public IP of the server.
# TURN/TURNS servers force all traffic through them.
webrtcICEServers2: []
  # - url: stun:stun.l.google.com:19302
  # if user is "AUTH_SECRET", then authentication is secret based.
  # the secret must be inserted into the password field.
  # username: ''
  # password: ''
  # clientOnly: false
# Maximum time to gather STUN candidates.
webrtcSTUNGatherTimeout: 5s
# Time to wait for the WebRTC handshake to complete.
webrtcHandshakeTimeout: 10s
# Maximum time to gather tracks.
webrtcTrackGatherTimeout: 2s

```

나중에 `sudo docker logs mediamtx --tail 20`명령어로 떠있는 mediamtx 컨테이너의 로그를 받아볼 수도 있겠다.

```jsonl
 
{"timestamp":"2026-08-03","level":"INF","message":"MediaMTX v1.19.3, linux, amd64"}
{"timestamp":"2026-08-03","level":"INF","message":"configuration loaded from /mediamtx.yml"}
{"timestamp":"2026-08-03","level":"INF","message":"[RTSP] started with listeners on :8554 (TCP/RTSP), :8000 (UDP/RTP), :8001 (UDP/RTCP)"}
{"timestamp":"2026-08-03,"level":"INF","message":"[RTMP] started with listener on :1935 (TCP/RTMP)"}
{"timestamp":"2026-08-03","level":"INF","message":"[HLS] started with listener on :8888 (TCP/HTTPS)"}
{"timestamp":"2026-08-03","level":"INF","message":"[WebRTC] started with listeners on :8889 (TCP/HTTPS), :8189 (UDP/ICE)"}
{"timestamp":"2026-08-03","level":"INF","message":"[SRT] started with listener on :8890 (UDP/SRT)"}
{"timestamp":"2026-08-03","level":"WAR","message":"[MoQ] certificate auto.key not found, generating it from scratch"}
{"timestamp":"2026-08-03","level":"INF","message":"[MoQ] started with listeners on :8892 (TCP/HTTP2), :8892 (UDP/HTTP3)"}
{"timestamp":"2026-08-03","level":"INF","message":"[RTSP] [conn x.x.x.x:xxxx] opened"}
{"timestamp":"2026-08-03","level":"INF","message":"[RTSP] [session xxxxx] created by x.x.x.x:xxxx"}
{"timestamp":"2026-08-03","level":"INF","message":"[path cam1] stream is available and online, 1 track (H264)"}
{"timestamp":"2026-08-03","level":"INF","message":"[RTSP] [session xxxxx] is publishing to path 'cam1'"}
{"timestamp":"2026-08-03","level":"INF","message":"[path cam1] RTP packets are too big (1460 > 1440), remuxing them into smaller ones"}

```

보기에 별 문제는 없어 보인다. (제일 마지막 줄은 물어보니까 알아서 쪼개는 과정임을 알리는 로그라고 한다.)

configuration을 적용하는 방법에는 여러가지가 있는데, 그 중 하나가 **실행중인 도커 컨테이너에 설정 파일을 옵션 인자로 넘겨주기**이다. 아래는 직역이다.

```bash

MediaMTX는 기본적으로 설정 파일(mediamtx.yml)을 포함해서 배포됩니다.

Docker 이미지 안에서는 루트 폴더(/mediamtx.yml)에 이 파일이 들어 있습니다.

이 설정 파일은 호스트의 파일로 덮어씌울 수 있습니다. 예를 들어:

bash
docker run --rm -it --network=host \   <--- host os의 포트와 ip 주소를 컨테이너도 그대로 쓰겠다는 뜻.
  -v "$PWD/mediamtx.yml:/mediamtx.yml:ro" \
  -p 8554:8554 \                 <------------ 예를 들면, 이건 rtsp publish용 포트
  -p 8889:8889 \                 <------------ 이건 webRTC용 (read용) 포트
  bluenviron/mediamtx:1

→ 이렇게 하면 현재 디렉토리($PWD)에 있는 mediamtx.yml을 컨테이너 내부의 /mediamtx.yml로 마운트해서, 내가 수정한 설정이 적용됩니다.

서버가 실행 중일 때도 설정 파일을 수정하면 자동으로 감지(hot reloading) 됩니다.

가능한 경우, 기존 클라이언트 연결을 끊지 않고 변경 사항을 적용합니다.

```

**참고**: `-v <호스트 경로>:<컨테이너 경로>:<옵션>` 으로, :ro는 read-only로 마운트한다는 것이다.

예로, `https`를 사용한 webRTC를 하려면,

```yaml

# mediamtx.yml 내용

# WebRTC (HTTPS/TLS)
webrtcEncryption: yes
webrtcServerCert: /etc/tailscale/server.crt  <-- 이것들은 tailscale에서 가져왔고,
webrtcServerKey: /etc/tailscale/server.key    <-- 모두 host os의 `/etc/tailscale/` 경로 아래에 있는 실제 certificate와 key이다.
                                                  마찬가지로 -v로 마운트 해주어야 한다. 아래와 같다.
                                                   -v /etc/ssl/tailscale:/etc/ssl/tailscale:ro   -e SSL_KEYSTORE_PASSWORD='your's'
# HLS (HTTPS/TLS)
hlsEncryption: yes
hlsServerCert: /etc/tailscale/server.crt
hlsServerKey: /etc/tailscale/server.key

```

으로 바꾸면 되겠다. host os의 mediamtx.yaml을 (-v로 마운트했기 때문에,) 컨테이너도 사용한다.
> 여기서는 `/etc/ssl/tailscale/`경로에 certificate와 key를 tailscale로부터 받아왔다.

이제 보안 설정이다.

##### 보안

Validate credentials

Credentials can be validated through one of these methods:

##### 1. Internal database: credentials are stored in the configuration file
##### 2. External HTTP server: an external HTTP URL is contacted for each authentication request
##### 4. External JWT provider: credentials are signed tokens released by an external identity server

JWT는 사실 어제 [애플코딩님의 영상](https://www.youtube.com/watch?v=XXseiON9CV0)을 하나 봤어서, 공부가 좀 필요하겠다 싶어서 놔두려고 한다.

추후 공부해서 따로 포스팅을 해두어야겠다.

아무튼 1번은 db에 사용자 이름과 비번을 저장해두는 방식이다.

```yaml

authInternalUsers:
  # Username. 'any' means any user, including anonymous ones.
  - user: any
    # Password. Not used in case of 'any' user.
    pass:
    # IPs or networks allowed to use this user. An empty list means any IP.
    ips: []
    # Permissions.
    permissions:
      # Available actions are: publish, read, playback, api, metrics, pprof.
      - action: publish
        # Paths can be set to further restrict access to a specific path.
        # An empty path means any path.
        # Regular expressions can be used by using a tilde as prefix.
        path:
      - action: read
        path:
      - action: playback
        path:

```

과 같은데, 문제는 **경고 문구**에 있었다.

인증 정보를 주고받게 되는데, RTSP같은 경우, WebRTC와 달리 TLS 같은 암호화가 필요하다는 뜻이다.
> 평문으로 주고받기 때문인데, 이러면 홈캠 영상인 미디어 패킷도 악의적인 공격자에게 노출될 위험이 있따.

다행히 이 부분은 TLS 인증서를 쓰거나, VPN을 쓰면 된다고 하는데, 내 경우에는 **tailscale**을 사용하기 때문에 VPN을 쓴다고 보면 된다.

이 db 방식의 user name과 password는 언제 들어가냐, 하면 아까 `rtsp`는 plain 으로 전송된다고 했으니까,

```

 rtsp://myuser:mypass@<server-ip>:8554/mystream

```

이렇게 url에 평문으로 넣어줘야 한다. 이래서 vpn이 꼭 필요했나보다.

`hls`나 `webRTC`에서 보낼 때는

```
Authorization: Basic base64(myuser:mypass)

```

와 같이 보내면 된다.

이떄, db에 저장할 비밀번호를 평문이 아닌, 

```bash

echo -n "mypass" | openssl dgst -binary -sha256 | openssl base64

```

로 만들어낸 base64의 sha256 변환 문자열을 저장할 수 있다. 그럴 때는,

```yaml

authInternalUsers:
  - user: sha256:j1tsRqDEw9xvq/D7/9tMx6Jh/jMhk3UfjwIB2f1zgMo=
    pass: sha256:BdSWkrdV+ZxFBLUQQY7+7uv9RmiSVA8nrPmjGjJtZQQ=
    permissions:
      - action: publish

```

와 같이 `sha256:`을 붙여줘야 한다.

여러 아이디, 혹은 특정 아이디 따로, 전체에게 따로 권한을 부여할 수 있다. 아래와 같다.

```yaml

 - user: any
    ips: []
    permissions:
      - action: read
        path: cam1          <----- /cam1 에서 '/' 빼야함.
  - user: usr1
    # Password. Not used in case of 'any' user.
    pass: sha256:xxxx
    # IPs or networks allowed to use this user. An empty list means any IP.
    ips: []
    # Permissions.
    permissions:
      # Available actions are: publish, read, playback, api, metrics, pprof.
      - action: publish
        # Paths can be set to further restrict access to a specific path.
        # An empty path means any path.
        # Regular expressions can be used by using a tilde as prefix.
        path: cam1 <---------------- publish랑 맞춰줘야 read할 수 있음.


```

위 예시는 any 에게 read 권한을 열어주고, mediaMTX 서버로 publish 할 수 있는 권한은 usr1에게만 줬다. 

단, 마찬가지로 WebRTC의 경우도 브라우저에서 쓰려면 https 인증서가 별도로 필요하다고 한다.
> tailscale의 도메인으로 접속시, 별도로 tailscale에서 https 인증서를 발급해준다고 한다.
> - 처음 알았는데, https 인증서는 도메인 단위지, ip 단위가 아니라고 한다.

https는 *tailscale*의 DNS 설정 페이지에서 https certificates를 enable한 뒤, 필요한 서버들(https로 접속해서 영상을 주고, 또 받을 각 서버)에 다운받아준다.
> 라즈베리파이는 웹 브라우저를 통해 전달하는 http 프로토콜이 아니기 때문에,(rtsp이다.) 웹 브라우저의 보안제약이 없어 https certificate를 다운받을 필요가 없다.

#### Proxy

proxy 내용은 stun 서버 내용이 아닌, cam 서버가 포트를 열어두면 mediaMTX 서버에서 **pull**해오는 방식이다.

사용자가 웹 브라우저로 접속 시 mediaMTX서버가 cam 서버로 stream을 pull해오겠단 건데, 원하는 방식은 아니기 때문에 넘어가겠다.

#### Log

```bash

docker run -v /var/log/mediamtx:/var/log/mediamtx myimage

```

도커 환경이므로, 이렇게 `-v`로 리눅스 서버 자체의 /var/log/ 경로를 마운트 해줘야 한다.
> - 알아서 로그를 일정기간 쌓아놓고, 또 잘라내는 *rotate* 기능을 해주는 관리 프로그램이 리눅스 서버 내에서 사용하기 때문이다.
> - 또, 도커 컨테이너가 내려갈 때마다 로그가 삭제되므로, 리눅스 서버 자체에 저장해야 보존할 수 있기 때문이다.

## 정리..?

이제 최종 정리를 해야겠다.

```

[ Tailscale 네트워크 (가상 프라이빗 도메인 제공) ]
                    │
            <my_server_name_>.<my_server_dns>.ts.net
                    │
                    ▼
[ 우분투 리눅스 서버 (호스트) ]  
  │
  ├─ 8080 포트 ──> [ Spring Boot 컨테이너 ] (웹 페이지 반환)
  │
  └─ 8889 포트 ──> [ MediaMTX 컨테이너 ]   (웹캠 영상 스트리밍)

```

이렇게 tailscale 덕분에 dns를 제공받은 뒤, https 인증서와 키 모두 발급받은 뒤, springboot server와 mediamtx 서버에 모두 적용했다.

그렇게 끝인 줄 알았으나, 문제가 생겼다.

## 문제발생

:8889로 접속하면 mediamtx의 configuration에 의해 팝업창으로 로그인창이 뜨고, mediamtx.yml에 적은 대로 아이디/비번을 치면 캠 화면을 볼 수 있다.

문제는 :8080으로 홈 대시보드에서 영상보기를 누르면 검은 화면만 나온다.

`f12` -> 콘솔창에서 다음과 같은 경고가 떴다.

```

Subresource requests whose URLs contain embedded credentials (e.g. `https://user:pass@host/`) are blocked. See https://www.chromestatus.com/feature/5669008342777856 for more details.

```

현재 query 형태로 (https://user:password를 쓴 뒤, @ 뒤에 도메인을 씀) 유저 아이디와 비번을 mediamtx에 넘겨줬는데, 이게 

팝업창이 아닌, \<iframe>의 경우는 subresource로서 보안 정책에 걸리는 문제이다.

AI 칭구들에게 물어본 결과 보통 "ngix의 역 프록시"를 이용한다고 한다. 아래 그림과 같다는데, 추후에 공부해보겠다.

```

Browser
     │
Spring
     │
Nginx
     │ (Authorization 추가)
MediaMTX

```

아무튼 지피티씨와 원만한 합의(?)를 내린 결과는 아래와 같다.

```

Browser
    │
    │ ① Spring 로그인 (JSESSIONID 또는 JWT)
    ▼
Spring Boot
    │
    │
    ├───────────────┐
    │               │
    ▼               ▼
 Dashboard      /api/mediamtx/auth
                    ▲
                    │ ③ authHTTP
                    │
              MediaMTX
                    ▲
                    │
             Raspberry Pi (RTSP Publish)

```

기본적으로 mediamtx는 `read`를 `any`로 열어놓고, spring에서 인증을 담당하게 한다는 것이다.

`mediamtx.yml`을 수정한다.

```yml

authmethod: internal --> http

authHTTPAddress: [도메인]/api/mediamtx/auth # 스프링 부트 쪽에서는 컨트롤러의 @postmapping()을 이용하면 되겠다.

```

아.., 공부할게 늘어났다. 

### JSESSIONID

**JSESSIONID**는 Java 기반의 웹 애플리케이션(예: Spring Boot, Tomcat, Servlet 기반 서버)에서 사용자의 로그인 상태나 세션(Session)을 식별하기 위해 발급하는 고유한 쿠키(Cookie) 이름이라고 한다.

젬나이의 설명은 아래와 같다.

```
JSESSIONID가 작동하는 방식

최초 접속 (로그인 요청):

  브라우저가 Spring Boot 웹 서버에 접속하거나 로그인을 시도합니다.

서버에서 세션 생성 & 발급:

  서버는 메모리에 해당 사용자를 위한 공간(세션)을 만들고, 무작위로 복잡한 고유 ID값(예: JSESSIONID=A1B2C3D4E5F6...)을 생성합니다.

  서버는 응답 헤더(Set-Cookie)에 이 ID를 실어서 브라우저에게 전달합니다.

쿠키 저장:

  브라우저는 전달받은 JSESSIONID를 자신의 쿠키 저장소에 보관합니다.

이후 모든 요청:

  브라우저가 다른 페이지(/home, /api/... 등)로 이동할 때마다, 요청 헤더(Cookie)에 JSESSIONID를 자동으로 포함해서 서버에 보냅니다.

서버의 사용자 식별:

  서버는 전달받은 JSESSIONID를 확인하고, "아! 아까 로그인했던 그 사용자가 맞구나!" 하고 인증된 상태로 응답을 내려줍니다.

```

아니 그냥 쿠키잖아? 

그렇다. 이미 새로고침해도 로그인 상태가 유지되기 때문에, `jsessionId`를 사용하고 있다.

**그렇다면 이 로그인시 브라우저의 `JSESSIONID`를 *mediamtx* 에서도 재사용할 수 있게 만들 순 없을까?**

**혹은 두 개를 따로 만들어야 하나?**

```
Browser
   |
   | JSESSIONID (Spring Security)
   |                |
Spring Boot :8080   |
                    |
MediaMTX            |    // 이게 안되나?
    |               |
    | authHTTP   <---
    |
Spring /api/mediamtx/auth

```

여기에 대해선 `JSESSIONID`가 뭔지부터 알아야 했다.

**"JSESSIONID = 서버에 저장된 세션 객체를 찾기 위한 열쇠"**이다.

이게 '*mediamtx*가 *Spring server*로 /api/mediamtx/auth를 `POST`로 날려도 JSESSIONID를 받아오지 못하는 이유'가 된다.

jsessionid를 소유한 주체는 *브라우저*이지 *SpringBoot 서버*가 아니기 때문이다.

예를 들어, 서버 메모리의 Session Storage (`SecurityContextRepository`라는 컴포넌트) 안에는 `jsessionid` 발급 후,

```

Session Storage

abc123
 |
 ├── username: kim
 ├── roles: USER
 └── SecurityContext
       |
       └── Authentication

```
다음과 같이 저장하고 있는데,

사용자가 http 요청과 함께 jsessionid를 붙여서 보내면, SecurityContext를 보고 확인 후, 요청을 통과시켜 처리한다.
> 로그인 사용자 '복원'이란 표현을 썼다.

**그렇다면** Spring을 시켜서 사용자 브라우저로부터 받아온다면? Spring을 중계하면 가능하다고 한다.
> 복잡한 proxy 설정이 있다고 하는데, 이건 추후로 미뤄둬야 겠다.
> **그냥 보내려 한다면, 도메인이 달라서 쿠키가 전달되지 않는다!**

하지만 이 방법은 곤란한게, webRTC의 특성 때문이다.

#### 1. 일반적인 웹 인증/리다이렉트 흐름  

  보통은 브라우저가 서버로 요청을 보내고, 서버가 인증 후 리다이렉트(예: 로그인 성공 후 특정 URL로 이동)하면 그걸로 끝. 
  > 이후에는 단순히 세션 기반 요청만 이어짐.

#### 2. WebRTC의 흐름

  WebRTC는 단순히 한 번의 요청으로 끝나는 게 아니라, 지속적인 미디어 스트리밍 연결을 유지해야 함.

  iframe 안에서 JavaScript가 동작하면서 `POST /whep` 같은 엔드포인트로 계속 요청을 보내고

  MediaMTX 같은 미디어 서버와 연결을 유지함.
  > 즉, 단순히 “리다이렉트 후 끝”이 아니라, 지속적인 데이터 교환과 연결 유지 과정이 뒤따릅니다.

그래서 추천받은 건 **별도의 인증 토큰**을 만들어서 mediamtx 단에서 저장 및 비교 조회를 해보는 것이다.
> 이때까지만해도 DB조회로 간단 인증이 가능하단 걸 몰랐다. auth HTTP에 대해 문서에 올라온 형식만 지켜서 건네주면 되는 거였다.

```

1. 사용자 로그인
        |
        v
2. Spring Security
        |
        v
3. Authentication.getName()
        |
        v
4. JWT 생성
   {
     username:"kim",
     path:"cam1",
     exp:5분
   }
        |
        v
5. iframe
   /cam1?user=JWT
        |
        v
6. MediaMTX authHTTP
        |
        v
7. Spring JWT 검증
        |
        v
8. 200 OK

```

아무튼 이제 토큰 방식으로 넘어간다.

### JWT (Json Web Token)

여기서 토큰이란, `JWT`이다. 안 쓰겠다 해놓고 어쩔 수 없이(?) 쓰게 됐다. 
> 인증은 꼭 jwt아니고, 단순 DB 조회도 가능하다고 하는데, 공부할겸 그냥 JWT로 가겠다.

위를 보면 사용자가 iframe을 통해서 `GET`요청을 mediamtx에게 보내고 있고,

그 다음, mediamtx에서는 인증 요청을 `POST`를 통해서 한다. 
> mediamtx.yml에 의해 
> - authHTTPAddress: https://domain-name:8080/api/mediamtx/auth 경로의 Spring 서버로 보내게 된다.

이때 mediamtx가 보내는 jwt(json 토큰)은 버전마다 정해진 규격이 있다. 예를 들면,

```json

{
  "ip": "100.x.x.x",
  "user": "",
  "password": "",
  "action": "read",
  "path": "cam1",
  "protocol": "http",
  "id": "xxxx"
}

```

그래서, 처음 사용자가 iframe으로 `GET`요청을 보낼 때 쿼리 스트링으로 날린 token을 그대로 복사해서 보내는 것이 아닌,

mediamtx의 규격 중 하나에 집어넣어서 보내게 된다.
>  "token": "<JWT>" 이런식으로 집어넣어서 보낸다고 한다.

공식 문서에 다음과 같이 authHTTP 형식이 기재돼있다.

```json

{
  "user": "user",
  "password": "password",
  "token": "token",
  "ip": "ip",
  "action": "publish|read|playback|api|metrics|pprof",
  "path": "path",
  "protocol": "rtsp|rtmp|hls|webrtc|srt",
  "id": "id",
  "query": "query",
  "userAgent": "userAgent"
}

```

여기에 맞춰 dto도 작성해주면 되겠다.

```java

@Getter
@Setter
public class MediaMtxJwtRequest {
	private String user;
  private String password;
  private String token;        // 이 토큰 부분을 무엇으로 할 거냐가 서버의 선택이다!
  private String ip;
  private String action;
  private String path;
  private String protocol;
  private String id;
  private String query;
  private String userAgent;	    
}

```




```java

@RequiredArgsConstructor
@RestController             // 리턴되는 자바객체를 json으로 변환.
@RequestMapping("/api/v1")
public class HomeApiController {
	
	private final MediaTokenService tokenService;
	
	@PostMapping("/mediamtx/auth")
	public ResponseEntity<Void> auth(
	        @RequestBody MediaMtxJwtRequest request
	) {

	    String token = request.getToken();      // token 키의 값을 가져온다.

	    if(tokenService.validate(token)) {
	        return ResponseEntity.ok().build();      // responseEntity는 reponse그 자체로, 응답을 ok(200)으로 할지말지 정할 수 있는 듯 하다.
	    }

	    return ResponseEntity.status(HttpStatus.UNAUTHORIZED) // unauthorized는 401이다.
	            .build();
	}

...

```

이런식으로 코드를 짰다. 간단히 jwt가 **tokenService**에서 검증받고, 유효하면 `ok`로 return하고, 그렇지 않으면 HttpStatus를 `401`로 설정한 후, return한다.


<details>
  <summary>(사실 embed 방식도 있지만)</summary>
<pre>
The iframe method is fit for most use cases, but it has some limitations:

it doesn’t allow to pass credentials (username, password or token) from the website to MediaMTX; credentials are asked directly to users.
</pre>
라고 한다. 해결하기 위해선 자바스크립트를 쓰는데, username이나 password가 노출된다. 탈락.
</details>

간단히 여기까지만 하고 [JWT](https://kau-newbie.github.io/JWT/)에 관한 포스팅으로 넘기겠다.
> 자세한 설명과 함께 실제 구현은 JWT에서 더 자세히 다루었다.

### authMethod

추가로, JWT방식을 쓰게 되면서, `authMethod`를 http로 바꿔주었는데, 이러면 기존 *rtsp*로 라즈베리파이가 스트리밍을 쏠 때 쓰던 internal과 충돌을 일으킨다.
> 진짜 몰라가지고 고생고생을 했다.

암튼 이때는 `authHttpExclude`를 설정해주면 된다.

```yml

authHTTPExclude:
  - action: api
  - action: metrics
  - action: pprof
  - action: publish

```

아무튼 이제 진짜진짜 마무리이다.

```

Browser
    │
    │ ① Spring 로그인 (JSESSIONID 또는 JWT)
    ▼
Spring Boot
    │
    │
    ├───────────────┐
    │               │
    ▼               ▼
 Dashboard      /api/mediamtx/auth
                    ▲
                    │ ③ authHTTP
                    │
              MediaMTX
                    ▲
                    │
             Raspberry Pi (RTSP Publish)

```

위의 아키텍처를 JWT를 통해 구현했다. 

1. 맨 처음, 클라이언트가 홈 화면으로 접속한다. 그 후, 로그인에 성공한다.
2. 홈 화면으로 url을 보냈으니 홈 컨트롤러가 작동하면서 jwt (인증토큰)을 만든다. 토큰은 Model에 넣어둔다.
3. View가 client가 접속할 home.html을 그릴 때, 앞서 만들어둔 jwt를 Model에서 꺼내서 <iframe>안에 url과 함께 넣는다.
4. 이러면 클라이언트(브라우저)가 home.html을 받으면서 jwt를 받게 된다.
5. 이후, 비디오 보기 버튼을 누르면, iframe을 통해 비디오를 보게 되는데, 이때 url을 통해 mediamtx 서버로 request가 jwt와 함께 전달된다.
6. mediamtx에서는 auth http (훅이라고 하는 것 같았다.) 설정으로 인해 spring에 위에서 언급한 json 형식으로 auth를 물어보게 되고(post형식),
7. spring에서는 api controller가 동작하면서 별도의 auth Service 컴포넌트에 의해 jwt를 검증하게 된다.
8. 검증 성공시 http 200을 반환시키게 되고 (responseEntity), mediamtx는 200을 보고 접속을 허용하게 된다.


끝.