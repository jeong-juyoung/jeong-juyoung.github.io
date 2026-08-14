---
layout: post
title: "Neural Network와 CNN, RNN, LSTM/GRU 그리고 Transformer로의 흐름"
date: 2026-08-14 18:00:00 +0900
categories: [dev, llm]
tags: [neural-network, CNN, RNN, LSTM, GRU, transformer, attention, deep-learning]
---

서종호(가시다)님의 **Hands-On LLM Serving and Optimization Study**를 진행하고 있다.

2주차 내용을 공부하면서 LLM의 핵심 구조인 **Transformer와 Attention**을 제대로 이해하려다 보니, 그 이전에 Neural Network가 어떻게 발전해 왔는지에 대한 기본적인 이해가 필요했다.

그래서 이번 글에서는 **Neural Network의 기본 개념부터 CNN, RNN, LSTM/GRU까지 간단히 살펴보고, 이러한 구조가 왜 Attention과 Transformer로 이어지게 되었는지** 흐름을 중심으로 정리해보려고 한다.

---

## Neural Network(뉴럴 네트워크)

- Neural Network(인공 신경망)는 인간의 신경망에서 아이디어를 얻어 만든 학습 구조로, 여러 개의 Neuron(뉴런)을 연결하여 데이터의 패턴을 학습한다.
- Neuron은 입력값에 Weight(가중치)와 Bias(편향)를 적용해 계산하고, 학습 과정에서 이 값들을 조정하면서 원하는 결과를 만들어낸다.
- MLP(Multilayer Perceptron)는 이러한 Neuron을 여러 Layer로 연결한 기본적인 신경망이다.

---

## 딥러닝의 주요 모델

- **DNN (Deep Neural Network)**
  - 여러 Hidden Layer로 구성된 기본적인 심층 신경망
  - 이미지 데이터를 처리할 때 다차원 데이터를 1차원으로 펼치는 `Flatten` 과정에서 공간적인 구조를 충분히 활용하기 어렵다는 한계가 있음

![DNN](/assets/img/for_post/2026-08-14-1.png)

- **CNN (Convolutional Neural Network)**
  - 이미지처럼 **공간적인 관계가 중요한 데이터**를 처리하는 데 특화
  - Convolution을 이용해 이미지의 특징을 추출
- **RNN (Recurrent Neural Network)**
  - 문장이나 시계열처럼 **순서가 중요한 데이터**를 처리하는 데 특화
  - 이전 정보를 Hidden State를 통해 다음 단계로 전달
- **Autoencoder**
  - 입력 데이터를 압축하여 중요한 특징을 학습한 뒤 다시 복원하는 구조

---

## Convolutional Neural Network (CNN)

- 이미지 처리에 주로 사용되는 **합성곱(Convolution) 기반 신경망**
- 이미지의 공간적인 구조를 유지하면서 **특징(Feature)을 추출**하는 데 강점이 있음
- 이미지 분류, 객체 탐지, 얼굴 인식 등 다양한 Computer Vision 분야에서 활용

### 이미지 데이터의 유형

CNN을 이해하기 전에 먼저 알아야 할 것은 **컴퓨터가 이미지를 어떻게 표현하는가**이다.

사람에게 이미지는 하나의 사진이지만, 컴퓨터에게 이미지는 결국 **픽셀(Pixel) 값으로 이루어진 숫자 배열**이다.

이미지의 종류에 따라 이 숫자 배열의 형태가 달라진다.

- **GrayScale**
  - 흑백 이미지
  - `가로 × 세로 × 1 Channel`
  - 하나의 픽셀을 `0 ~ 255` 값으로 표현
  - `0`에 가까울수록 검정, `255`에 가까울수록 흰색
- **Color (RGB)**
  - `가로 × 세로 × 3 Channel`
  - Red, Green, Blue의 **3개 채널**로 구성
  - 각 채널은 `0 ~ 255` 값을 가짐
  - 예: `(255, 0, 0)` → 빨간색
- **Binary**
  - 각 픽셀을 `0` 또는 `1`로 표현
  - 흑/백 두 가지 상태만 표현

### CNN 의 구조

![CNN 구조](/assets/img/for_post/2026-08-14-2.png)

- CNN은 기존 신경망에 **Convolution Layer와 Pooling Layer를 활용하여 이미지의 특징을 먼저 추출한 뒤**, 최종적으로 Fully Connected Layer 등을 통해 분류 등의 작업을 수행하는 구조로 이해할 수 있다.
- 초기 Layer에서는 선이나 모서리 같은 **단순한 특징**을 찾고, Layer가 깊어지면서 형태나 객체의 일부처럼 **더 복잡한 특징**을 학습한다.

### Convolutional Layer

- Convolutional Layer의 역할은 이미지에서 **특징(Feature)을 추출하는 것**이다.
- 작은 크기의 **Filter(Kernel)**가 이미지 위를 이동하면서 연산하고, 특정 패턴이 존재하는 위치를 찾아낸다.

![Convolutional Layer](/assets/img/for_post/2026-08-14-3.png)

```
설명
1. 입력 이미지에서 3*3 영역을 가져온다.
2. Filter의 각 값과 입력 값을 곱한다.
3. 모든 값을 더한다.
4. 이 값을 Feature Map의 한 픽셀로 기록한다.
5. Filter를 오른쪽(또는 아래)로 한 칸 이동하여 반복한다.
```

- 예를 들어 어떤 Filter는 **세로선**, 다른 Filter는 **가로선이나 모서리** 등에 반응하도록 학습될 수 있다.
- **Filter / Kernel**
  - 이미지의 특징을 찾기 위한 작은 크기의 가중치 배열
  - 학습 과정에서 Filter의 값이 학습됨
  - 여러 Filter를 사용하면 서로 다른 특징을 추출할 수 있음
- **Feature Map**
  - Filter가 입력 이미지 전체를 이동하며 계산한 결과
  - 하나의 Filter가 하나의 출력 Feature Map을 생성
  - Filter가 `K개`라면 출력 Channel도 `K개`
- **Stride**
  - Filter가 한 번에 이동하는 간격
  - Stride가 커질수록 Feature Map의 가로·세로 크기가 작아짐
- **Padding**
  - 입력 이미지의 가장자리에 값을 추가하는 것
  - 가장자리 정보 손실을 줄이고 출력 크기를 조절하는 데 사용
  - Zero Padding은 주변을 `0`으로 채우는 방식
- **ReLU**
  - Convolution 연산 후 주로 사용하는 활성화 함수
  - 비선형성을 추가하여 복잡한 특징을 학습할 수 있도록 함

### Pooling layer

- Pooling은 Convolution을 통해 만들어진 **Feature Map의 크기를 줄이는(Downsampling) 과정**이다.
- 보통 `2 × 2` 또는 `3 × 3` 크기를 사용하며, Feature Map의 크기를 줄이기 위해 `Stride = 2`를 사용하는 경우가 많다. 일반적으로 크기를 줄이는 것이 목적이므로 Padding은 사용하지 않는 경우가 많다.

![Pooling layer](/assets/img/for_post/2026-08-14-4.png)

- **Max Pooling**
  - 일정 영역에서 가장 큰 값을 선택
  - 강하게 나타난 특징을 유지
- **Average Pooling**
  - 일정 영역의 평균값을 사용

#### 왜 크기를 줄일까?

- CNN에서 Convolution을 반복하면 많은 Feature Map을 처리해야 하기 때문에 계산량도 커진다.
  - **계산량 감소**: 처리해야 하는 데이터가 줄어듦
  - **중요한 특징 유지**: Max Pooling의 경우 강하게 반응한 특징을 남김
  - **작은 위치 변화에 덜 민감**: 특징이 조금 이동해도 비슷한 결과를 얻을 수 있음
  - 이후 Layer에서 더 넓은 영역의 정보를 효과적으로 다룰 수 있음

---

## Recurrent Neural Network (RNN, 순환 신경망)

![RNN](/assets/img/for_post/2026-08-14-5.png)

- RNN은 **순서(Sequence)가 있는 데이터를 처리하기 위해 만들어진 신경망**
- 이전 시점의 정보를 **Hidden State**에 저장하고 다음 시점의 입력과 함께 사용
- 자연어, 음성, 시계열 데이터처럼 **순서와 앞뒤 관계가 중요한 데이터**에 활용됨

### RNN은 왜 필요할까?

일반적인 DNN은 각각의 입력을 **독립적으로 처리**한다.

하지만 문장처럼 순서가 중요한 데이터에서는 이전 정보가 현재 데이터를 이해하는 데 영향을 준다.

**EX) 예시**

![RNN 예시](/assets/img/for_post/2026-08-14-6.png)

- RNN의 핵심은 **Memory 역할을 하는 Hidden State**에 있다.
- **Hidden State**
  - Hidden State는 **이전까지 처리한 Sequence의 정보를 전달하는 값**이다.

### RNN 구조

- RNN은 현재 입력 `xₜ`뿐만 아니라 이전 시점의 Hidden State `hₜ₋₁`도 함께 사용한다.

![RNN 구조](/assets/img/for_post/2026-08-14-7.png)

- 이를 시간 순서대로 펼쳐보면 다음과 같다.

![RNN 시간 순서 펼치기](/assets/img/for_post/2026-08-14-8.png)

- 각 RNN Cell이 서로 다른 모델처럼 보이지만, 실제로는 **동일한 Weight를 모든 시점에서 공유**한다.

#### 문장 예시

**EX ) "나는 어제 영화를 보았다"**

![문장 예시](/assets/img/for_post/2026-08-14-9.png)

- **각 시점의 Hidden State `hₜ` 는 이전까지 본 단어들의 정보를 요약해서 담고 있다.**
- 문장 예시를 시간 흐름도로 표현

![시간 흐름도](/assets/img/for_post/2026-08-14-10.png)

### RNN의 유형

- RNN은 입력과 출력 Sequence의 형태에 따라 여러 방식으로 사용할 수 있다.

![RNN 유형](/assets/img/for_post/2026-08-14-11.png)

- **One-to-One**: 하나의 입력 → 하나의 출력
- **One-to-Many**: 하나의 입력 → 여러 개의 출력
- **Many-to-One**: 여러 입력 → 하나의 출력
  - 예: 문장 감정 분석
- **Many-to-Many**: 여러 입력 → 여러 출력
  - 예: 번역, Sequence Labeling
- 자연어 처리에서는 문장이 **Token의 Sequence**이기 때문에 RNN이 많이 사용되었다.

### RNN의 한계

- **장기 의존성(Long-Term Dependency) 문제**
  - Sequence가 길어질수록 초반의 정보를 끝까지 유지하기 어려움
  - 먼 과거의 정보가 현재 출력에 중요한 경우 제대로 반영하기 어려울 수 있음
- **기울기 소실(Vanishing Gradient)**
  - RNN은 시간 순서대로 연결된 구조를 따라 역전파하며 학습함
  - 이 과정이 길어지면 Gradient가 점점 작아져 **앞쪽 시점까지 학습 신호가 제대로 전달되지 않을 수 있음**
  - 장기 의존성 문제가 발생하는 주요 원인 중 하나
- **순차 처리로 인한 병렬화의 어려움**
  - `h₁`을 계산해야 `h₂`, `h₂`를 계산해야 `h₃`를 계산할 수 있음
  - 각 Token을 순서대로 처리해야 하므로 **병렬 연산에 제약이 있고 긴 Sequence 처리에 비효율적**

> 이러한 RNN의 한계를 개선하기 위해 **LSTM과 GRU**가 등장했고, 이후 **Attention과 Transformer**로 발전하게 된다.

### 보완 모델 (LSTM / GRU)

![LSTM / GRU](/assets/img/for_post/2026-08-14-12.png)

### LSTM (Long Short-Term Memory)

- RNN의 장기 의존성 문제를 개선하기 위해 만들어진 구조
- **Cell State**라는 별도의 정보 전달 경로를 사용하여 오래된 정보를 유지
- **Gate**를 이용해 어떤 정보를 기억하고, 버리고, 출력할지 결정
- LSTM에는 대표적으로 세 가지 Gate가 있다.
  - **Forget Gate**: 이전 정보 중 무엇을 버릴지 결정
  - **Input Gate**: 새로운 정보 중 무엇을 기억할지 결정
  - **Output Gate**: 저장된 정보 중 무엇을 다음 단계의 출력으로 사용할지 결정

### GRU (Gated Recurrent Unit)

- GRU는 LSTM과 마찬가지로 **RNN의 장기 의존성 문제를 개선하기 위한 구조**이다.
- LSTM보다 구조를 단순화하여 두 개의 Gate를 사용한다.
  - **Update Gate**: 이전 정보를 얼마나 유지하고 새로운 정보를 얼마나 반영할지 결정
  - **Reset Gate**: 이전 정보 중 얼마나 무시할지 결정

---

## 왜 Attention과 Transformer로 넘어갔을까?

- RNN의 장기 의존성 문제를 개선하기 위해 **LSTM과 GRU**가 등장했지만, 여전히 데이터를 순서대로 처리해야 한다는 구조적인 한계가 있었다.
  - 이전 시점의 계산이 끝나야 다음 시점을 처리할 수 있어 **병렬 처리에 제약**이 있음
  - Sequence가 길어질수록 멀리 떨어진 정보 사이의 관계를 처리하기 어려움
  - LSTM/GRU는 장기 의존성 문제를 개선했지만 **순차 처리라는 RNN의 구조 자체는 그대로 유지**
- Transformer는 Attention으로 이 둘을 동시에 해결
  - 모든 토큰이 다른 모든 토큰을 한 번에 참조 → 병렬 처리 가능
  - 거리와 무관하게 관계를 직접 계산 → 장기 의존성 해결
- 전체 흐름
  - 신경망(기본) → CNN(공간) / RNN(순서) → 한계 → Attention → Transformer → GPT·BERT → LLM Serving

---

## 참조

- 참조한 영상 링크
  - [https://www.youtube.com/watch?v=Dp2Ph3v8V-E](https://www.youtube.com/watch?v=Dp2Ph3v8V-E)
  - [https://www.youtube.com/watch?v=zpMHGnSAusI](https://www.youtube.com/watch?v=zpMHGnSAusI)
- 참조한 블로그
  - [https://nanini.tistory.com/81](https://nanini.tistory.com/81#google_vignette)
  - [https://technical-support.tistory.com/65](https://technical-support.tistory.com/65)
  - [https://hackernoon.com/what-is-an-rnn-recurrent-neural-network-in-deep-learning](https://hackernoon.com/what-is-an-rnn-recurrent-neural-network-in-deep-learning)
