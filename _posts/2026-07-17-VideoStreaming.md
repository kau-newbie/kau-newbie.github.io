---
layout: post
title:  "[project] video-streaming 공부"
author: kau-newbie
categories: [project, video, streaming]
image: assets/images/forPost/videostreaming/videoStreamingExample.png
---

요약: video streaming이란 무엇인가

# Video Streaming 공부

프로젝트를 진행하다보니, 홈캠으로부터 비디오 스트리밍을 받아올 필요가 있었다.

rasberypi 3 a+라는, 충분하다면 충분하고 모자라다면 모자랄 미니 컴퓨터를 통해 비디오 스트리밍을 쏘려고 하니,

제미나이가 webRTC라는 기술을 추천해줬다. 그게 뭔데? 참을 수 없었다. 그래서 공부를 시작하려 한다.

## 제미나이의 로드맵

당장 1도 몰라 막막한데, webRTC는 전문서적이 인터넷에 쳐보니 국내서적으로 1권 나와있었다.

이건, 그냥 인터넷으로 하는게 좋을 것 같다 생각이 들었다.

그래도, 로드맵 정도는 있으면 좋겠다 싶어서 제미나이에게 짜달라고 했다. 아래와 같다.

---

### 🗺️ 미디어 스트리밍 전문가 로드맵

```
[1단계: 스트리밍 기초 & 코덱] ──> [2단계: FFmpeg 마스터] ──> [3단계: WebRTC & NAT 트래버설] ──> [4단계: 파이프라인 통합]
 (프로토콜/코덱 이해)               (영상 처리/변환/송출)             (ICE/STUN/TURN/Signaling)          (MediaMTX/라즈베리파이/Web)

```

#### 1단계: 미디어 스트리밍 기초 & 프로토콜 이해

영상/음성 데이터가 어떻게 압축되고 네트워크를 통해 전달되는지 기본 원리를 파악하는 단계입니다.

* **핵심 학습 주제:**
* **코덱 (Codec):** H.264, H.265 (비디오) / AAC, OpUS (오디오) — 압축 알고리즘 및 I/P/B 프레임 개념
* **컨테이너 (Container):** MP4, MKV, FLV — 비디오+오디오+자막 메타데이터 포맷
* **스트리밍 프로토콜 비교:**
* RTSP (Real-Time Streaming Protocol - CCTV/IP카메라 표준)
* HLS (HTTP Live Streaming - 유튜브/넷플릭스 등 HTTP 기반 스트리밍)
* WebRTC (Web Real-Time Communication - 초저지연 P2P 통신)


#### 2단계: FFmpeg 마스터하기

FFmpeg는 모든 멀티미디어 처리의 "스위스 아미 나이프"입니다. 입출력 스트림 제어와 CLI 명령어를 익히는 단계입니다.

* **핵심 학습 주제:**
* FFmpeg 기본 명령어 구조: `ffmpeg [input 옵션] -i [입력] [output 옵션] [출력]`
* 하드웨어 인코딩/디코딩: 라즈베리파이(`h264_v4l2m2m`), NVIDIA(`nvenc`), Intel(`qsv`)
* 필터 그래프 사용법 (`-vf`): 해상도 변경, 자막 입히기, 오디오 믹싱
* RTSP/RTMP/HLS 프로토콜 변환 및 가상 테스트 스트림 생성 (`testsrc`)


#### 3단계: WebRTC & NAT 트래버설 깊이 파기

웹 브라우저에서 딜레이 없이 실시간으로 영상/음성을 주고받는 WebRTC 심화 단계입니다.

* **핵심 학습 주제:**
* **시그널링 (Signaling) & SDP:** 브라우저와 서버 간 연결 규격(Offer/Answer) 교환
* **ICE Candidate & NAT Traversal (핵심!):** 사설 IP 환경에서 상대방 IP/포트를 찾는 과정
* **STUN & TURN 서버:** P2P 직통로 형성(STUN) 및 방화벽 우회 중계 서버(TURN/Coturn)
* **WHEP (WebRTC HTTP Egress Protocol):** HTTP REST API로 WebRTC 미디어 수신을 주고받는 현대적 표준 프로토콜

#### 4단계: 실전 파이프라인 구축 & 서비스 통합

개념들을 묶어서 라즈베리파이(엣지 디바이스)부터 우분투(미디어 서버), 웹 대시보드까지 완성하는 단계입니다.

* **핵심 학습 주제:**
* 라즈베리파이 카메라 제어 (`Picamera2`) 및 파이썬-FFmpeg 프로세스 연동
* MediaMTX 설정 최적화 (`mediamtx.yml` 커스텀)
* HTML5 `<video>` 태그 및 JavaScript `RTCPeerConnection` / WHEP Client 라이브러리 연동
* 서비스 확장: OpenCV 연동을 통한 실시간 모션 감지 및 AI 객체 인식 추가


### 📚 추천 학습 링크 & 공식 문서

#### 1. WebRTC 전문 학습 자료

* **[HTML5 Rocks - Getting Started with WebRTC](https://www.html5rocks.com/en/tutorials/webrtc/basics/)**
* 구글 개발자들이 직접 작성한 WebRTC의 가장 기본적이면서 완벽한 입문 아티클입니다.


* **[WebRTC Weekly & Blog by Tsahi Levent-Levi (bloggeek.me)](https://bloggeek.me/)**
* 전 세계 WebRTC 업계에서 가장 유명한 전문가의 블로그입니다. NAT 트래버설, STUN/TURN, SFU/MCU 개념이 완벽하게 정리되어 있습니다.


* **[W3C WebRTC 1.0 공식 명세 및 MDN 문서](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API)**
* JavaScript에서 `RTCPeerConnection`, `getUserMedia` 등을 다룰 때 참조하는 MDN 공식 기술 문서입니다.



#### 2. FFmpeg 학습 자료

* **[FFmpeg Official Documentation](https://ffmpeg.org/documentation.html)**
* 공식 문서로, 모든 옵션과 필터에 대한 상세 설명이 들어있습니다.


* **[FFmpeg Wiki (H.264 Encoding Guide)](https://trac.ffmpeg.org/wiki/Encode/H.264)**
* H.264 코덱의 프리셋(`ultrafast`, `veryfast` 등), 비트레이트 설정, CRF 값 조절 등 실무 가이드가 잘 나와 있습니다.



#### 3. 미디어 서버 및 라이브러리 (MediaMTX & Picamera2)

* **[MediaMTX GitHub (bluenviron/mediamtx)](https://github.com/bluenviron/mediamtx)**
* README 및 Wiki 문서에 RTSP, WebRTC(WHEP), HLS 설정을 위한 환경변수와 샘플이 매우 깔끔하게 다뤄져 있습니다.


* **[Raspberry Pi Picamera2 Official Manual](https://www.google.com/search?q=https://datasheets.raspberrypi.com/picamera2/picamera2-python-manual.pdf)**
* 라즈베리파이 재단에서 공식 제공하는 Python 카메라 제어 라이브러리 `Picamera2`의 PDF 메뉴얼입니다.


### 💡 학습 진행 팁

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

그림으로 보면 아래와 같겠다.

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

[기가 막힌 영상을 하나 찾았다.](https://www.youtube.com/watch?v=MqP8ur-FNuA) 몹시 전체적으로 잘 설명해주는 것 같다.
> 나도 이걸 보고 공부해서, 정말 그런지는 확인을 못해주겠다.

[기가 막힌 영상을 또 하나 찾았다.](https://www.youtube.com/watch?v=-4NXxY4maYc) 포맷과 컨테이너에 대해 설명해준다.

아무튼, 영상들을 정리해보겠다.

[infra-frame관련 영상](https://www.youtube.com/watch?v=DljGCnNzkag)


