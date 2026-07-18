---
layout: post
title:  "[ns-3]ns-3 간단히 알아보기"
author: kau-newbie
categories: [cc, ns-3]
image: assets/images/forPost/ns-3icon.png
---


## 개요

ns-3 라는 시뮬레이터는 공모전을 준비하면서 알게되었다. 

대략적으로 하나의 시뮬레이션은,

`NW를 nw layer에 따라 설정` -> `노드를 생성 후 nw 설정을 주입` -> `그 후 시뮬레이터를 run()`하는 순서를 가진다.


## 폴더구조

폴더구조에 따라 코드를 짜고, 빌드 후 실행까지 이곳저곳을 옮겨다녔다.

폴더 구조는 아래와 같다.

(젬나이의 도움을 받아....)

```
ns-allinone-3.xx/
└── ns-3.xx/                
    ├── scratch/             ← [코드 작성] 내가 작성한 시뮬레이션 예제 코드를 두는 곳
    ├── src/                 ← [내부 소스] ns-3의 기본 네트워크 모듈(LTE, Wi-Fi 등)이 있는 곳
    ├── contrib/             ← [기여 모듈] 외부 사용자들이 개발한 모듈이 들어가는 곳
    ├── build/               ← [빌드 결과물] 컴파일된 실행 파일과 라이브러리가 저장되는 곳
    ├── ns3                  ← [컴파일/실행 도구] CMake 기반의 실행 스크립트 (과거의 ./waf 역할)
    └── ...

```
1. `scratch/` 안에서 코드를 짠다. (.cc 파일)
2. `ns-3.xx` 폴더로 나온다. 그 후, `./ns3 build` 명령어를 친다.
- scratch/ 폴더 안의 내용 변경 사항을 알아서 감지해서 컴파일을 진행합니다.
3. 계속해서 `ns-3.xx`위치에서 `./ns3 run scratch/my-simulation` 명령어를 입력해 실행한다.
- parameter인자를 cmd에서 넘기려면 ./ns3 run "scratch/my-simulation --nNodes=10 --simTime=20" 와 같이 치면 된다.
4. 실행결과를 `ns-3.xx` 위치에서 확인하면 된다.(주로 .xml파일)


예를 들면, `/ns-allinone-3.47/ns-3.47/examples/tutorial/first.cc` 를 복붙해서

`/ns-allinone-3.47/ns-3.47/scratch$` 아래에 `basic-twosides.cc`라는 새 프로그램을 하나 만든다.

그 후, 2, 3, 을 거쳐 4번처럼 ns-3.xx 위치에서 ls 명령어를 입력하면, 

```bash

user@DESKTOP:~/contest/ns-allinone-3.47/ns-3.47$ ls
AUTHORS               NrDlTxRlcStats.txt           UlMacStats.txt                drone-test-b.xml
CHANGES.md            NrUlMacStats.txt             UlPathlossTrace.txt           drone_log.txt
CMakeLists.txt        NrUlPdcpRxStats.txt          UlPdcpStats.txt               drone_sim_log.txt
CODE_OF_CONDUCT.md    NrUlPdcpStatsE2E.txt         UlRlcStats.txt                examples
CONTRIBUTING.md       NrUlPdcpTxStats.txt          UlRxPhyStats.txt              hexagonal-topology.gnuplot
DlCtrlSinr.txt        NrUlRlcRxStats.txt           UlSinrStats.txt               lte-animation-test.xml
DlDataSinr.txt        NrUlRlcStatsE2E.txt          UlTxPhyStats.txt              my-animation-test.xml
DlMacStats.txt        NrUlRlcTxStats.txt           VERSION                       ns3
DlPathlossTrace.txt   README.md                    __pycache__                   pyproject.toml
DlPdcpStats.txt       RELEASE_NOTES.md             bindings                      scratch
DlRlcStats.txt        RxPacketTrace.txt            build                         setup.cfg
DlRsrpSinrStats.txt   RxedGnbMacCtrlMsgsTrace.txt  build-support                 setup.py
DlRxPhyStats.txt      RxedGnbPhyCtrlMsgsTrace.txt  cmake-cache                   src
DlTxPhyStats.txt      RxedUeMacCtrlMsgsTrace.txt   contrib                       test.py
LICENSE               RxedUePhyCtrlMsgsTrace.txt   debug_UlSchedulingTest        utils
LICENSES              RxedUePhyDlDciTrace.txt      doc                           utils.py
NrDlMacStats.txt      TxedGnbPhyCtrlMsgsTrace.txt  drone-animation-test-v00.xml  wifi-animation-test.xml
NrDlPdcpStatsE2E.txt  TxedUeMacCtrlMsgsTrace.txt   drone-animation-test-v01.xml
NrDlPdcpTxStats.txt   TxedUePhyCtrlMsgsTrace.txt   drone-animation-test00.xml
NrDlRlcStatsE2E.txt   UlInterferenceStats.txt      drone-output.tr

```

여러 xml 파일들이 존재하는 것을 확인할 수 있다. (설정하기 나름에 따라 txt파일 등등이 가능하다. `netanim`을 이용해 애니메이션 실행도 가능하다.)

## basic-twosides.cc 코드 예시

```cpp

/*
 * SPDX-License-Identifier: GPL-2.0-only
 */
#include "ns3/netanim-module.h"
#include "ns3/applications-module.h"
#include "ns3/core-module.h"
#include "ns3/internet-module.h"
#include "ns3/network-module.h"
#include "ns3/point-to-point-module.h"

// Default Network Topology
//
//       10.1.1.0
// n0 -------------- n1
//    point-to-point
//

using namespace ns3; // cpp에서는 package처럼 namespace란 걸 쓴다.
                     // 이때, ::는 클래스(static var, method)를 나타내거나 혹은 namespace를 나타낼 때 붙인다.
                     // 단, 객체에는 보통 언어들처럼 . 를 붙인다.

NS_LOG_COMPONENT_DEFINE("FirstScriptExample"); // 로그에 태그를 붙이는 과정. COMPONENT가 이 파일을 의미.
                                               // 태그 이름을 "First..."로 한 것.

int
main(int argc, char* argv[])
{
    CommandLine cmd(__FILE__); // 인자를 받겠다는 의미. ./ns3 스크립트를 쓰는 것이기 때문에, cmd에서는
                               // 인자를 ./ns3 run "basic_p2p.cc --nNodes=10" 형태로 줘야 한다. 뒷 부분
                               // 의 "string"을 ns3 라는 스크립트에게 넘겨주는 것(파이썬).
                               // __FILE__은 이 파일을 의미한다.
    cmd.Parse(argc, argv);     // 밑에서 cmd.로 하는 인자가 있으면 parse에 의해 알아서 들어감.

    Time::SetResolution(Time::NS); //시간 단위 설정
    LogComponentEnable("UdpEchoClientApplication", LOG_LEVEL_INFO); // 미리 정의된 UdpEcho...App...를 로그 생성
                                                                    // 을 할 수 있는 상태로 만듦.
    LogComponentEnable("UdpEchoServerApplication", LOG_LEVEL_INFO); // 마찬가지. LOG_LEVEL_INFO는 역시 그 정도.

    NodeContainer nodes; // NodeContainer가 빈 통신 주체 컨테이너.
    nodes.Create(2);     // 두 개 만듦. 아직은 텅 빔.

    PointToPointHelper pointToPoint; // 미리 설정됨. -Helper는 설정을 쉽게 해주는 클래스. PointToPoint 통신 외에도
                                     // Wifi에 대한 helper들(WifiHelper, Ya..머시기WifiHelper 등등, 그리고 Bus에 여러 노드들이 달려있는 토폴로지를 나타내는 통신도 있다.
    pointToPoint.SetDeviceAttribute("DataRate", StringValue("5Mbps"));
    pointToPoint.SetChannelAttribute("Delay", StringValue("2ms"));

    NetDeviceContainer devices; // 장치(NIC)
    devices = pointToPoint.Install(nodes); // nodes 인스턴스들에 NIC꽂고 pointToPoint라는 channel에 연결해서 devices라는 NetDevice 컨테이너에 담음.

    InternetStackHelper stack; // internet stack = windows, linux에 들어있는 nw sw bundle? 그런거.
                               // L3, L4, ARP/ICMP 등 기능이 들어있다.
                               // 랜카드는 전기를 주고받는 물리적 장치일 뿐, 실제 프로토콜 선택은 OS가 한다.
                               // 따라서 노드(컴퓨터)에다가이sw bundle인 stack을 쥐어준다.
    stack.Install(nodes);

    Ipv4AddressHelper address; // ipv4address의 helper 클래스
    address.SetBase("10.1.1.0", "255.255.255.0"); // 10.1.1.x 로 나눠주겠다.

    Ipv4InterfaceContainer interfaces = address.Assign(devices); // net device container인 devices를 이 address에 assign.
                                                                 // 각 랜카드에 address를 할당해준다
                                                                 // 그 결과를 interfaces라는 container에 저장.
                                                                 // 나중에 클라이언트가서버 ip 물어볼 때도 이 interface container에 물어본다고 함.

    UdpEchoServerHelper echoServer(9); // 실제로 데이터를 주고받을 앱 서버를 만듦. udp echo server임.
                                       // 인자는 포트 번호. 9번 포트를 염. echo server를여는 것.
                                       //
                                       // 항상
                                       // 1. helper 객체 만든다.
                                       // 2. 세부 설정을 한다.
                                       // 3. 실행해서 결과물을 container에 담는다.

    ApplicationContainer serverApps = echoServer.Install(nodes.Get(1)); //9번포트 설정된 서버 헬퍼를 이용해서
                                                                        //1번 노드에 실제 서버 앱을 설치
    serverApps.Start(Seconds(1));   // 시뮬레이션 시간 1초에 start - > 10초에 stop
    serverApps.Stop(Seconds(10));

    UdpEchoClientHelper echoClient(interfaces.GetAddress(1), 9);// 클라이언트는 설정이 조금 더 많음.
                                                                // des ip와 des port, 즉 des 소켓 주소를 알아야 함.
                                                                // GetAddress()의 인자는 노드 번호임.
    echoClient.SetAttribute("MaxPackets", UintegerValue(1));      // max packet 수를 1로.
    echoClient.SetAttribute("Interval", TimeValue(Seconds(1)));   // (여러 개 보낼 때) 1초 간격으로 보냄.
    echoClient.SetAttribute("PacketSize", UintegerValue(1024));   // 패킷 크기는 1024 B

    ApplicationContainer clientApps = echoClient.Install(nodes.Get(0)); // 0번 노드에 클라이언트 앱을 설치 및
                                                                        // 시작.(2초) -> 10초에 종료.
    clientApps.Start(Seconds(2));
    clientApps.Stop(Seconds(10));

    AnimationInterface anim("my-animation-test.xml"); // 2. XML 파일 이름 설정
    anim.SetConstantPosition(nodes.Get(0), 10.0, 10.0); // 3. 노드 위치 지정 (x, y)
    anim.SetConstantPosition(nodes.Get(1), 50.0, 10.0);
    Simulator::Run();
    Simulator::Destroy();
    return 0;
}

```






