---
layout: post
title:  "[project]AndroidTutor(9) Gemma4:e2b Fine Tunning-1"
author: kau-newbie
categories: [ gemma4, project, Ai, finetunning]
image: assets/images/prj-androidtutor-basicmodel.jpg
---

# gemma4 파인튜닝

## transformer

시작은 RNN.

seq2seq모델은 encoder에서 `은닉 상태`(loop를 돌며 매 입력단위를 읽을 때마다 나오는 중간 결과에 tanh()등을 씌운 값.)를 최종적으로 모아 `context vector`라는 고정된 크기의 벡터를 뽑아낸다.

문제는 context vector는 '고정된 크기'라는 점이다. 단 세 입력 단위 문장을 읽어들일 때도 이 전체 context vector를 사용한다.

또, seq2seq모델은 입력이 길어질수록 앞 순서의 입력 단위가 0에 수렴한다고 한다.(매 입력층에 이전 입력 상태 $h_{t-1}$가 들어간다. 이때, tanh는 최대값 1이기 때문에 매번 곱해질 때마다 0에 수렴해간다.)

따라서 앞 문장은 뒷 문장에 영향을 크게 끼치지 못한다. 입력이 커지면 커질수록 성능이 저하된다.

이를 해결하기 위해 attention구조가 나왔다.

여기까지를 [동빈나님의 교육동영상](https://www.youtube.com/watch?v=AA621UofTUA)을 보면, 다음과 같이 정리하고 있다.

[문제 상황] 
- 하나의 문맥 벡터가 소스 문장의 모든 정보를 가지고 있어야 하므로 성능이 저하됩니다.

[해결 방안]
- 그렇다면 매번 소스 문장에서의 출력 전부를 입력으로 받으면 어떨까요?
    - 최신 GPU는 많은 메모리와 빠른 **병렬 처리**를 지원합니다.

물론, attention 역시 context vector를 쓰는 건 마찬가지이다. decoder가 사용할 context vector는 그대로이다. 즉, 고정된 크기의 은닉상태를 받게된다.

다만, 매 은닉 상태마다 나온 모든 은닉 상태를 모은 뒤 이걸 인코더에서 디코더로 보낸다. 이때! **가중 평균**을 사용하여 전체 context vector 중에 어느 부분에 집중해야할지(attention) 디코더에게 알려줄 추가 정보를 준다. (context vector는 디코더가 가중평균 정보를 이용해 그때그때 만든다고 한다.)

|구분|기존 Seq2seq (Vanilla)|Attention 결합형 Seq2seq|
|---|---|---|
|인코더가 넘겨주는 것|맨 마지막 시점의 은닉 상태 벡터 딱 1개 ($h_{last}$)|모든 시점의 은닉 상태 벡터 '전부 다' ($h_1, h_2, \dots, h_T$)|
|Context Vector의 생성 주체|인코더가 문장을 다 읽고 미리 만들어 둠|디코더가 매 시점 필요한 부분을 직접 계산해서 생성함Context Vector의 |
|특징|문장이 길어지든 짧아지든 항상 고정된 내용|디코더가 단어를 뱉을 때마다 내용이 계속 바뀌는 동적 벡터문장|
| 길이에 따른 성능|긴 문장일수록 정보를 잃어버림 (병목 현상)|문장이 길어져도 필요한 정보를 찾아가므로 성능 유지|

### seq2seq with Attention: Decoder

Energy: 

$$e_{ij}=a(S_{i-1},h_{j})$$

Weight(가중치):

$$\alpha = \frac{exp(e_{ij})}{\sum_{k=1}^T_{x} exp(e_{ik})}$$





















RNN(Recurrent Neural Network)과 FC(Fully Connected Layer, 전결합층)는 데이터를 처리하는 방식과 구조에서 근본적인 차이가 있습니다.

1. Fully Connected Layer (FC)
독립적인 처리: FC는 입력 데이터 간의 관계를 시계열 순서나 흐름으로 보지 않습니다. 입력된 모든 노드가 출력 노드와 각각 연결되어, 들어오는 데이터(벡터)를 독립적으로 계산합니다.
고정된 크기: 입력과 출력의 크기가 고정되어 있으며, 정적인 데이터를 처리하는 데 적합합니다(예: 이미지 분류의 마지막 단).
2. RNN (Recurrent Neural Network)
순차적인 정보 유지: RNN은 이름처럼 **'순환'**하는 구조를 가집니다. 시퀀스 데이터(시간 순서가 있는 데이터, 예: 언어, 주가)를 처리할 때, 이전 단계에서 계산된 히든 스테이트(Hidden State) 값을 다음 단계의 입력으로 다시 활용합니다(3:56-4:18).
맥락 파악: 이전까지 입력된 단어들의 정보를 기억하고 현재 입력과 결합하기 때문에, 문장의 맥락이나 데이터의 흐름을 파악하는 데 특화되어 있습니다.
요약 비교
FC: 데이터 간의 독립성을 전제로 하며, 전체 데이터의 특징을 정적으로 추출할 때 사용합니다.
RNN: 데이터의 **'순서(Order)'**를 중요하게 생각하며, 과거의 정보를 기억해 현재의 결과에 반영합니다(1:17-1:25).