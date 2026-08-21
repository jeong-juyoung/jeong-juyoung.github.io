---
layout: post
title: "Attention과 Transformer 구조 정리"
date: 2026-08-21 15:00:00 +0900
categories: [dev, llm]
tags: [transformer, attention, self-attention, encoder, decoder, positional-encoding]
---

서종호(가시다)님의 **Hands-On LLM Serving and Optimization Study**를 진행중이다.

스터디 대비 진도가 좀 느리긴 하지만 처음 공부하는 부분이라 천천히 나가는 중이다..!

---

- **Attention**은 인코더와 디코더 사이에서 어떤 입력 정보를 중요하게 볼지 결정하는 가중치다.
  - 디코더가 출력 단어를 만들 때 인코더의 입력 단어들을 참고한다.

  ![Attention](/assets/img/for_post/2026-08-21-1.png)

- **Self-Attention**은 하나의 인코더 또는 디코더 내부에서 각 단어가 같은 문장의 다른 단어를 얼마나 중요하게 참고할지 결정하는 가중치다.
  - 문장 안의 각 단어가 같은 문장에 있는 다른 단어들을 참고한다.

  ![Self-Attention](/assets/img/for_post/2026-08-21-2.png)

- Encoder는 인풋
- Decoder는 아웃풋

---

## Transformer 구조

- Transformer는 **Encoder–Decoder 구조**로 구성된다.
- Encoder와 Decoder는 각각 동일한 구조의 블록을 여러 개 쌓아 만든다.
  - **Encoder**
    - Multi-Head Self-Attention
    - Feed-Forward Network
  - **Decoder**
    - Masked Multi-Head Self-Attention
    - Encoder–Decoder Attention
    - Feed-Forward Network

![Transformer 구조 1](/assets/img/for_post/2026-08-21-3.png)

![Transformer 구조 2](/assets/img/for_post/2026-08-21-4.png)

Transformer에는 세 가지 Attention이 사용된다.

1. **Encoder Self-Attention**
   - Encoder 내부에서 입력 문장의 각 단어가 다른 입력 단어와의 관계를 계산한다.
   - 입력 문장의 문맥을 파악하는 역할을 한다.
2. **Masked Decoder Self-Attention**
   - Decoder 내부에서 이전에 생성된 단어들의 관계를 계산한다.
   - 아직 생성되지 않은 미래 단어를 참고하지 못하도록 Attention을 제한한다.
3. **Encoder–Decoder Attention**
   - Decoder가 다음 단어를 생성할 때 Encoder의 출력 정보를 참고한다.
   - Decoder의 Query와 Encoder의 Key·Value를 사용한다.

---

## Transformer의 입력 처리 과정

### Embedding

- 단어(token) 형태의 데이터를 수치로 변환
  - 초기에는 one-hot vector 형태로 입력되며, embedding layer를 통해 학습
  - 유사 단어는 유사한 값을 지니도록 embedding 수행

![Embedding](/assets/img/for_post/2026-08-21-5.png)

- Inputs을 나는 학생이다 라는 자연어를 넣는다 가정
  - → 자연어는 인간이 이해할 수 있는 형태의 정보
  - → 기계가 이해할 수 있게 바꿔줘야함 Encoding 한다. embedding 한다라고 말함

### Positional Encoding

- 단어 사이 순차성을 반영하기 위한 기법
  - RNN계열 방법론과는 달리 입력값을 순차적으로 처리하지 않음(병렬적 처리)
  - 순차성을 부여할 수 있도록 추가적인 처리가 필요 → positional encoding

  ![Positional Encoding](/assets/img/for_post/2026-08-21-6.png)

  - vector 안에는 숫자들이 포함되어있다.
- 주기함수를 활용하여 positional encoding 구성
  - Sinusoid positional encoding
  - 순차성을 부여하되 의미 정보가 변질되지 않도록 -1~1 사이를 반복
  - 모든 단어는 같은 차원의 positional encoding 벡터
  - 먼 단어사이에는 큰 값이, 가까운 단어사이에는 작은 값이 오도록 구성

> Embedding과 Positional Encoding은 일반적인 데이터 전처리라기보다, 자연어 입력을 Transformer가 처리할 수 있는 형태로 만드는 **입력 표현 단계**에 해당한다.

---

## Encoder

### Self-Attention

- 문장에 있는 모든 단어의 관계를 비교하여 문맥이 반영된 새로운 특징 벡터 Z를 만든다.
  - 각 단어의 Embedding Vector에 학습 가능한 가중치 행렬을 적용하여 Query, Key, Value Vector를 생성한다.
  - 현재 기준 단어의 Query와 문장에 있는 모든 단어의 Key를 비교하여 Attention Score를 계산한다.
  - Attention Score를 Softmax에 적용하여 각 단어를 얼마나 참고할지 나타내는 Attention Weight로 변환한다.
  - Attention Weight를 각 단어의 Value Vector에 곱하고 모두 더하여 최종 특징 벡터 Z를 만든다.

![Self-Attention 계산](/assets/img/for_post/2026-08-21-7.png)

Self-Attention은 문장에 있는 모든 단어의 관계를 비교하고, 다른 단어의 정보가 반영된 새로운 벡터를 만드는 과정이다.

예를 들어 `나는 학생이다`라는 문장이 입력되면, 먼저 각 단어의 Embedding Vector로부터 Query, Key, Value Vector를 만든다.

- **Query:** 현재 단어를 기준으로 다른 단어와의 관계를 확인할 때 사용한다.
- **Key:** 각 단어가 어떤 특징을 가지고 있는지 나타낸다.
- **Value:** 해당 단어가 실제로 가지고 있는 정보를 담는다.

#### 단어 사이의 관계 계산

그림에서는 `나는`이라는 단어를 기준으로 다른 단어와의 관계를 계산한다.

먼저 `나는`의 Query와 문장에 있는 모든 단어의 Key를 각각 비교한다.

- `나는`의 Query와 `나는`의 Key 비교
- `나는`의 Query와 `학생`의 Key 비교
- `나는`의 Query와 `이다`의 Key 비교

그림에서는 비교 결과로 각각 136, 116, 64가 나온다. 이 숫자는 `나는`을 처리할 때 각 단어를 얼마나 중요하게 볼 것인지 판단하기 위한 점수다.

점수의 크기를 조정한 뒤 Softmax 함수를 적용하면 다음과 같은 가중치가 만들어진다.

- `나는`: 0.952
- `학생`: 0.047
- `이다`: 0.000

전체 가중치의 합은 1이 된다. 이 예시에서는 `나는`을 처리할 때 자기 자신을 가장 중요하게 참고하고, `학생`은 조금 참고하며, `이다`는 거의 참고하지 않는다.

![단어 관계 계산 결과](/assets/img/for_post/2026-08-21-8.png)

#### 새로운 벡터 생성

Softmax를 통해 구한 Attention Weight를 각 단어의 Value Vector에 곱한다. 가중치가 클수록 해당 단어의 정보를 더 많이 반영하고, 가중치가 작을수록 적게 반영한다.

`나는`을 기준으로 구한 가중치를 `나는`, `학생`, `이다`의 Value Vector에 각각 적용한 뒤, 그 결과를 모두 더한다. 이렇게 만들어진 결과가 `나는`의 새로운 특징 벡터인 Z1이다.

Z1에는 `나는` 자체의 정보뿐만 아니라 `학생`, `이다`와의 관계도 함께 반영된다. 같은 과정을 `학생`과 `이다`에도 반복하면 각각 Z2, Z3가 만들어진다.

#### Multi-Head Attention 결과 결합

- 8개의 Head 결과를 연결한 뒤, 출력 가중치 W₀로 입력과 같은 차원에 맞춘다.

![Multi-Head Attention 결합](/assets/img/for_post/2026-08-21-9.png)

앞에서 살펴본 Z1, Z2, Z3 생성 과정은 하나의 Attention Head에서 이루어지는 계산이다. Transformer는 이러한 Attention을 여러 Head에서 서로 다른 가중치를 사용하여 동시에 수행한다.

그림의 예시에서는 8개 Head가 만든 결과를 하나로 이어 붙이면 3×16 크기가 된다. 하지만 입력 Embedding의 크기는 3×4이므로, 다음 단계로 전달하기 위해 출력도 3×4 크기로 맞춰야 한다.

이를 위해 연결된 결과에 출력 가중치 행렬 `W₀`를 적용한다. 그 결과 입력과 동일한 3×4 크기의 최종 특징 벡터 Z가 만들어진다. `W₀`의 값도 모델의 학습 과정에서 결정된다.

정리하면 Self-Attention은 다음 순서로 동작한다.

1. 각 단어에서 Query, Key, Value를 만든다.
2. 현재 단어의 Query와 모든 단어의 Key를 비교한다.
3. 비교 점수를 Softmax를 통해 가중치로 변환한다.
4. 가중치를 각 단어의 Value에 적용한다.
5. Value들을 합쳐 문맥이 반영된 새로운 벡터를 만든다.

#### 최종 정리하자면..

![Self-Attention 최종 정리](/assets/img/for_post/2026-08-21-10.png)

`Thinking Machines`라는 자연어 문장이 입력되면 먼저 Token을 Embedding Vector로 변환하고 위치 정보를 더한다. 이렇게 만들어진 입력 벡터를 X라고 한다.

Transformer는 하나의 관점으로만 단어 관계를 확인하지 않고, 8개의 Attention Head를 사용해 서로 다른 관점에서 관계를 계산한다. 따라서 각 Head에는 자신만의 Query, Key, Value 가중치 행렬이 있다.

각 Head에서는 입력 X에 Query, Key, Value 가중치 행렬을 각각 적용하여 Query, Key, Value Vector를 만든다. 즉, 8개의 Head에서 서로 다른 Query, Key, Value가 만들어진다.

이후 각 Head 내부에서 다음 과정이 진행된다.

1. Query와 Key를 비교하여 단어 사이의 Attention Score를 구한다.
2. Attention Score를 통해 각 단어를 얼마나 참고할지 결정한다.
3. 해당 가중치를 Value Vector에 적용한다.
4. 가중치가 적용된 Value Vector들을 모두 더해 해당 Head의 결과 Z를 만든다.

8개의 Head가 같은 과정을 수행하면 Z0부터 Z7까지 총 8개의 결과가 만들어진다. 이 결과들을 옆으로 이어 붙인 다음, 출력 가중치 행렬 `W₀`를 적용한다.

그 결과 입력과 동일한 형태의 최종 출력 Z가 만들어진다. 그림에서는 입력 문장에 Token이 2개이고 각 Token을 4개의 숫자로 표현하므로, 최종 출력도 2×4 크기가 된다.

정리하면 다음과 같다.

**자연어 입력 → Embedding과 위치 정보 추가 → 8개 Head에서 각각 Query·Key·Value 생성 → Head별 Self-Attention 계산 → Z0부터 Z7 생성 → 모든 결과 연결 → W₀ 적용 → 최종 출력 Z 생성**

### Feed Forward

![Feed Forward](/assets/img/for_post/2026-08-21-11.png)

Self-Attention을 통해 각 단어에는 문맥이 반영된 새로운 특징 벡터가 만들어진다. 이 벡터들은 다음으로 Feed Forward Network를 통과한다.

Feed Forward Network는 각 단어의 특징 벡터를 개별적으로 처리하는 작은 신경망이다. `나는`, `학생`, `이다`의 벡터가 서로 섞이지 않고 각각 동일한 신경망을 통과한다.

Self-Attention이 단어 사이의 관계를 파악하는 과정이라면, Feed Forward Network는 각 단어에서 얻은 특징을 한 번 더 변환하고 가공하는 과정이다.

#### Feed Forward Network가 필요한 이유

Self-Attention에서는 Query, Key, Value에 사용되는 가중치 행렬을 곱해 단어 사이의 관계를 계산한다. 그러나 이러한 선형적인 계산만 반복하면 복잡한 특징을 충분히 표현하기 어렵다.

Feed Forward Network에서는 다음과 같은 처리를 수행한다.

1. 입력 벡터에 첫 번째 선형 변환을 적용하여 차원을 확장한다.
2. 활성화 함수를 적용하여 비선형성을 추가한다.
3. 두 번째 선형 변환을 적용하여 원래 차원으로 되돌린다.

비선형성을 추가하면 단순한 직선 형태의 관계뿐만 아니라 더 복잡한 특징과 패턴을 학습할 수 있다.

같은 Encoder 블록 안에서는 모든 단어가 동일한 Feed Forward Network의 가중치를 공유한다. 다만 각 단어의 입력 벡터가 다르므로 출력 결과도 서로 다르게 만들어진다. 또한 Encoder 블록이 달라지면 Feed Forward Network도 서로 다른 가중치를 사용한다.

Feed Forward Network의 출력에는 입력 벡터를 다시 더하는 Residual Connection을 적용하고, 이후 Layer Normalization을 수행한다. 이를 통해 기존 정보를 유지하면서 학습을 안정적으로 진행할 수 있다.

### Encoder : Self-Attention Example

- Input : The animal didn't cross the street because **`it`** was too tired

![Self-Attention Example](/assets/img/for_post/2026-08-21-12.png)

- `it` 이라는 단어를 인코딩할 때 어떨 때는 `animal`에 가장 집중하고, 어떨 때는 `tired`에 집중하는 방식임
- `it`이라는 단어 표현에 '동물'과 '피곤하다'의 표현이 일부 섞여 있음

---

## Decoder

### Masked Self-Attention

- 현재시점까지 주어진 단어사이 관계 고려
  - 미래 시점에 대해 마스킹 하는 이유 : 모델은 순차에서 선행하는 단어에만 중요도를 부여하여 이전 예측 단어를 기반으로 올바른 예측 단어를 생성하도록 학습

Encoder가 한국어 입력 문장을 처리했다면, Decoder는 영어 출력 문장을 생성한다. 학습 과정에서 Decoder는 정답 문장을 한 칸 이동시킨 형태로 입력받고, 앞에 있는 단어를 이용해 다음 단어를 예측한다.

이때 아직 생성되지 않은 미래 단어를 미리 참고하면 정답을 알고 예측하는 것과 같아진다. 따라서 Mask를 적용하여 현재 위치보다 뒤에 있는 단어에는 Attention을 주지 못하게 한다.

예를 들어 `I am a student`에서 `I`를 처리할 때는 `I`만 볼 수 있으며, 뒤에 있는 `am`, `a`, `student`는 볼 수 없다. `am`을 처리할 때는 `I`와 `am`까지 볼 수 있다.

이처럼 Decoder의 Masked Self-Attention은 현재 단어까지의 정보만 사용하도록 제한하여, 앞에서부터 단어를 하나씩 생성하는 방법을 학습하게 한다.

![Masked Self-Attention](/assets/img/for_post/2026-08-21-13.png)

### Encoder-Decoder Attention

- 마지막 Encoder의 Key,Value values를 활용하여 encoding 정보 고려

Masked Self-Attention을 거친 Decoder의 벡터는 다음으로 Encoder–Decoder Attention을 통과한다. 이 과정에서는 Decoder가 다음 단어를 생성할 때 Encoder가 처리한 입력 문장의 어떤 부분을 중요하게 참고할지 결정한다.

Query는 **Decoder**의 Masked Self-Attention 결과에서 가져오고, **Key와 Value는 Encoder**의 최종 출력에서 가져온다. 따라서 같은 문장 내부에서 관계를 계산하는 Self-Attention과 달리, Encoder와 Decoder 사이의 관계를 계산한다.

그림에서는 Decoder의 `am`과 Encoder가 처리한 `나는`, `학생`, `이다`의 관계를 비교한다. Attention Weight를 계산한 결과 `이다`가 가장 높은 값을 얻는다. 따라서 `am`의 특징 벡터를 만들 때 `이다`의 정보를 가장 많이 반영한다.

이처럼 Encoder–Decoder Attention은 Decoder가 출력 단어를 생성할 때 원본 입력 문장의 관련 정보를 찾아 연결하는 역할을 한다.

![Encoder-Decoder Attention](/assets/img/for_post/2026-08-21-14.png)

### Feed Forward

- Encoder와 마찬가지로, Attention을 통해 얻은 각 단어의 특징 벡터를 Feed Forward Network에서 개별적으로 가공한다.

### Prediction

Decoder에서 처리된 최종 출력 벡터는 Softmax 과정을 거쳐 각 Token에 대한 확률로 변환된다. Prediction 모듈은 이 확률을 바탕으로 다음에 올 Token을 예측한다. 그림에서는 `I am` 다음에 올 Token으로 `a`가 선택된다.

![Prediction](/assets/img/for_post/2026-08-21-15.png)

---

## Transformer 실험 결과

Transformer는 영어–독일어와 영어–프랑스어 기계 번역 실험에서 기존 모델보다 높은 BLEU 점수를 기록하며 당시 최고 수준의 성능을 달성했다.

순차적으로 계산하는 기존 모델과 달리 병렬 연산이 가능해 학습에 필요한 연산량도 크게 줄였다. 또한 특정 번역 작업에만 종속되지 않고, 다른 문제에도 적용할 수 있는 범용적인 구조임을 확인했다.

![실험 결과 1](/assets/img/for_post/2026-08-21-16.png)

![실험 결과 2](/assets/img/for_post/2026-08-21-17.png)

---

## Transformer 응용 사례

Transformer는 구성 요소와 전체 구조를 목적에 맞게 변경하여 다양한 문제에 활용할 수 있다.

- **Module-level:** Attention 방식, Positional Encoding, 활성화 함수, Feed Forward Network 등의 세부 모듈을 작업에 맞게 변경한다.
- **Architecture-level:** Encoder와 Decoder의 연결 방식이나 전체 구조를 변경하여 다양한 형태의 Transformer를 구성한다.
- **Pre-Trained Models:** Encoder만 사용하거나 Decoder만 사용하는 등 목적에 맞는 사전 학습 모델을 만든다.
- **Applications:** 자연어뿐만 아니라 이미지, 음성, 멀티모달 등 다양한 데이터와 문제에 적용한다.

Transformer는 번역 모델로 시작했지만, 유연한 구조와 높은 확장성을 바탕으로 다양한 인공지능 모델의 기반 아키텍처로 활용되고 있다.

![응용 사례](/assets/img/for_post/2026-08-21-18.png)

---

## Summary

- **RNN 기반 방법론**
  - 입력값을 순차적으로 전달받아 처리한다.
  - 순차적으로 전달되는 정보를 통해 데이터의 순서와 시계열 정보를 반영할 수 있다.
  - 앞의 계산이 끝나야 다음 계산을 수행할 수 있어 병렬 처리가 어렵고, 연산 효율이 낮다는 한계가 있다.
- **Transformer**
  - 입력값을 순차적으로 받지 않고 한 번에 처리한다.
  - 병렬 처리가 가능하여 연산 측면에서 효율적으로 활용할 수 있다.
  - Positional Encoding을 적용하여 입력 데이터의 순서 정보를 반영한다.
  - Self-Attention을 통해 단어 사이의 관계와 중요한 정보를 반영한다.
- **Transformer의 활용**
  - 구성 요소와 전체 구조를 변경하여 다양한 문제에 적용할 수 있다.
  - 자연어 처리: 기계 번역, 문서 요약, 질의응답
  - 이미지 처리: 이미지 인식, 객체 탐지
  - 제조 분야: 상태 분류, 이상 탐지

---

## 참고자료

- [https://medium.com/@hari4om/word-embedding-d816f643140](https://medium.com/@hari4om/word-embedding-d816f643140)
- [https://jalammar.github.io/illustrated-transformer/](https://jalammar.github.io/illustrated-transformer/)
- [https://arxiv.org/abs/2106.04554](https://arxiv.org/abs/2106.04554)
- [https://arxiv.org/abs/1706.03762](https://arxiv.org/abs/1706.03762)

## 참고영상

- [https://www.youtube.com/watch?v=a_-YgMO0u0E](https://www.youtube.com/watch?v=a_-YgMO0u0E)
