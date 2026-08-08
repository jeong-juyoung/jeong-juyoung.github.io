---
layout: post
title: "LLM Serving and Optimization - Chapter 2. Large Language Model Serving"
date: 2026-08-08 12:00:00 +0900
categories: [dev, llm]
tags: [LLM, transformer, KV-cache, vLLM, attention, serving, inference]
---

*서종호(가시다)님의 Hands-On LLM Serving and Optimization Study* 을 진행중이고, 책 내용을 읽으며 정리한 내용이다.

---

## Inside the Mind of a Transformer

언어 모델의 개요

![언어 모델 개요](/assets/img/for_post/2026-08-08-chater2-1.png)

- **Word2Vec**

  2013년 Word2Vec이 등장하면서 단어를 **Vector 형태로 표현(Embedding)** 할 수 있게 되었다.
  이를 통해 단어 사이의 의미적 관계를 수치적으로 표현할 수 있게 되었다.

- **RNN**

  언어처럼 순서가 중요한 데이터를 처리하기 위해 RNN(Recurrent Neural Network)이 사용되기 시작했다.

  RNN은 이전 정보를 다음 단계로 전달하면서 문맥을 처리할 수 있었지만 다음과 같은 한계가 있었다.

  - 데이터를 순차적으로 처리해야 함
  - GPU를 이용한 병렬 처리가 어려움
  - 문장이 길어질수록 오래된 정보를 유지하기 어려움

  이를 개선하기 위해 LSTM과 GRU가 등장했지만 순차 처리라는 근본적인 문제는 여전히 존재했다.

- **Transformer**

  2017년 Google의 **"Attention Is All You Need"** 논문에서 Transformer가 등장했다.

  Transformer는 RNN의 순차적인 구조 대신 **Self-Attention과 Positional Encoding**을 사용한다.

  이를 통해

  - 입력을 병렬로 처리할 수 있고
  - 멀리 떨어진 단어 사이의 관계(Long-range Dependency)를 효과적으로 처리할 수 있게 되었다.

  **Transformer의 등장 이후 대표적으로 BERT와 GPT 계열이 발전했다.**

- **BERT**

  Transformer의 **Encoder**를 기반으로 하며, 문맥을 양방향으로 처리한다.

  주로 문장의 의미를 이해하는 작업에 적합하다.

  - Text Classification
  - Contextual Embedding

- **GPT**

  Transformer의 **Decoder**를 기반으로 하며, 앞의 Token들을 바탕으로 **다음 Token을 예측하는 방식**으로 텍스트를 생성한다.

  ```
  이전 Token들
      ↓
  GPT
      ↓
  다음 Token 예측
      ↓
  반복
      ↓
  문장 생성
  ```

  모델의 Parameter와 학습 데이터 규모가 증가하면서 GPT 계열의 성능도 크게 향상되었고, Zero-shot이나 Few-shot처럼 별도의 추가 학습 없이 다양한 작업을 수행하는 능력도 발전했다.

  **이러한 대규모 Pre-trained Language Model을 일반적으로 LLM(Large Language Model)이라고 부른다.**

---

## The Autoregressive Nature of Transformers

LLM은 텍스트를 한 번에 생성하지 않고 **한 번에 하나의 Token씩 순차적으로 생성**한다. 새로운 Token은 이전 Prompt와 지금까지 생성된 모든 Token을 기반으로 예측된다.
이 과정을 **Autoregressive Generation**이라고 한다. 생성은 최대 길이에 도달하거나 EOS(End-of-Sequence)와 같은 종료 Token이 생성될 때까지 반복된다.

![Autoregressive Generation](/assets/img/for_post/2026-08-08-chater2-2.png)

---

## Decoder-Only Transformer Architecture

현재 GPT, Llama, Qwen과 같은 대부분의 Generative LLM은 **Decoder-Only Transformer Architecture**를 사용한다.

Transformer 계열에는 여러 구조가 있지만, 이 책에서는 LLM Serving과 가장 관련이 높은 Decoder-Only 구조에 집중한다.

- **Why Demonstrate Decoder-Only Transformer Architecture?**
  - Transformer 기반 모델은 공통적으로 여러 Transformer Block과 Self-Attention을 사용하지만 구조와 용도는 서로 다르다.
  - Decoder-Only Transformer는 GPT, Llama, Qwen처럼 **텍스트 생성 작업에 널리 사용되는 구조**이기 때문에 이 책에서 주요 대상으로 다룬다.

### Model architecture

Decoder-Only Transformer는 크게 다음 세 부분으로 구성된다.

![Decoder-Only 모델 구조](/assets/img/for_post/2026-08-08-chater2-3.png)

#### Step 1: Tokenizer and embedding

Tokenizer는 입력 문장을 모델이 처리할 수 있는 형태로 변환한다.

- Text를 Token으로 분리
- Token을 Token ID로 변환
- Token ID를 Embedding Vector로 변환

즉, 사람이 읽는 문장을 모델이 연산할 수 있는 숫자 형태로 바꾸는 과정이다.

#### Step 2: Transformer (decoder) blocks

여러 개의 Decoder Block을 순차적으로 통과하면서 각 Token의 문맥 정보를 계산한다.

Decoder Block의 출력은 **Hidden State**이며, 입력 Token들의 문맥이 반영된 Vector 표현이다.

일반적으로 마지막 Token의 Hidden State가 다음 Token을 예측하는 데 사용된다.

#### Step 3: Language modeling (LM) head

Transformer Block의 Hidden State를 Vocabulary의 각 Token에 대한 점수(Logit)로 변환한다.

이 점수를 기반으로 다음에 생성할 Token을 선택한다.

```
Hidden State
→ Vocabulary Logits
→ Token별 확률
→ 다음 Token 선택
```

---

## Transformer (decoder) block

![Transformer Decoder Block](/assets/img/for_post/2026-08-08-chater2-4.png)

Decoder Block은 LLM에서 대부분의 연산이 수행되는 핵심 부분이다.

주요 구성 요소는 다음과 같다.

- **Self-attention layer**
  - 입력 Token 사이의 관계를 계산하여 각 Token이 어떤 다른 Token을 중요하게 참고해야 하는지 결정한다.
  - 이를 통해 Token의 의미를 주변 문맥에 맞게 이해할 수 있다.
- **Feedforward neural network (FFN)**
  - Attention에서 얻은 문맥 정보를 기반으로 각 Token의 표현을 추가로 변환한다.
  - 학습 과정에서 얻은 지식을 활용하여 다음 예측에 사용할 더 풍부한 Token Representation을 생성한다.
  - Decoder Block의 구조를 이해하면 향후 Attention Kernel, CUDA 최적화 등 Serving 성능 개선 방법을 이해하는 데 도움이 된다.

---

## Capture Token Context by Calculating Attention

Self-Attention은 각 Token이 이전 Token들을 참고하여 자신의 문맥을 이해하도록 한다.

예를 들어 `US capital city`에서 `capital`이라는 Token은 `US`와의 관계를 참고하여 금융의 capital이 아니라 **수도**라는 의미로 이해할 수 있다.

#### Attention calculation

각 Token에서는 세 가지 Vector가 만들어진다.

- Query (Q)
- Key (K)
- Value (V)

![Attention Calculation](/assets/img/for_post/2026-08-08-chater2-5.png)

Query와 다른 Token의 Key를 비교하여 **Attention Score**를 계산하고, 이를 기반으로 어떤 Token을 얼마나 중요하게 참고할지 결정한다.

그 결과 Value들을 가중 합산하여 문맥 정보가 반영된 새로운 Token Representation을 만든다.

#### Multi-head attention

하나의 Attention만 계산하는 것이 아니라 여러 개의 Attention Head를 동시에 사용한다.

각 Head는 서로 다른 Q/K/V를 사용하여 Token 사이의 다양한 관계를 병렬로 학습한다.

예를 들어 각각의 Head가 다음과 같은 관계에 집중할 수 있다.

- 문법적 관계
- 위치 관계
- 의미적 관계

각 Head의 결과를 결합하여 최종 Attention 결과를 만든다.

![Multi-head Attention](/assets/img/for_post/2026-08-08-chater2-6.png)

- **You Just Need to Grasp the Transformer Concept—Not the Math**

  LLM Serving 관점에서는 Attention의 복잡한 수학을 깊게 이해할 필요는 없다.

  중요한 것은 다음과 같다.

  - Attention은 많은 연산을 필요로 한다.
  - 특히 Prefill 단계에서 연산량이 크다.
  - 입력 Sequence가 길어질수록 Memory와 Latency 비용이 증가한다.
  - 이러한 특성이 KV Cache와 Serving Optimization의 필요성으로 이어진다.

---

## Executing LLM Generation: A Step-by-Step Walkthrough

LLM이 실제로 Token을 생성하는 과정을 코드로 확인한다.

이 과정에서 LLM Serving의 핵심 개념인 다음 내용을 다룬다.

- Token 단위 실행
- Prefill
- Decode
- KV Cache

실제 Production에서는 직접 이러한 Inference Loop를 구현하지 않고 Serving Framework를 사용하지만, 내부 동작을 이해하기 위해 기본 실행 과정을 살펴본다.

#### Run the Qwen Model

`Hugging Face pipeline`을 이용하면 Qwen 같은 LLM을 간단하게 실행할 수 있다.

`pipeline`은 Tokenization, Model Loading, Generation 등의 내부 과정을 추상화하여 간단한 API로 Text Generation을 수행할 수 있도록 한다.

#### Model Prediction, Line by Line

LLM 내부에서는 다음 과정을 반복한다.

```
현재 Input Sequence
→ Model 실행
→ 다음 Token 후보 생성
→ Token 선택
→ 기존 Input에 Token 추가
→ 다시 Model 실행
```

![Model Prediction](/assets/img/for_post/2026-08-08-chater2-7.png)

각 단계에서 Vocabulary 전체에 대한 Logit을 계산하고 확률 분포를 만든 뒤 다음 Token을 선택한다.

KV Cache를 사용하지 않는 경우 새로운 Token이 추가될 때마다 **기존 전체 Sequence를 다시 계산**한다.

따라서 Sequence가 길어질수록 매 Token 생성에 필요한 연산량과 시간이 증가한다.

![Model Prediction 2](/assets/img/for_post/2026-08-08-chater2-8.png)

![Model Prediction 3](/assets/img/for_post/2026-08-08-chater2-9.png)

#### Enable the KV Cache to Boost Performance

KV Cache는 이전 Token들의 Attention 계산 결과인 **Key와 Value를 저장하여 재사용하는 방식**이다.

KV Cache를 사용하지 않으면 새로운 Token을 생성할 때마다 기존 모든 Token의 Attention을 다시 계산해야 한다.

KV Cache를 사용하면

```
이전 Token의 K/V → Cache에서 재사용
새 Token → 새롭게 계산
```

하는 방식으로 동작한다.

![KV Cache](/assets/img/for_post/2026-08-08-chater2-10.png)

이를 통해 중복 연산을 크게 줄이고 Token Generation 속도를 향상할 수 있다.

대신 이전 Token의 K/V를 저장해야 하므로 **Memory 사용량이 증가한다.**

즉, KV Cache는 **Memory를 추가로 사용하는 대신 Compute 비용을 줄이는 최적화 방식**이다.

책의 예제에서는 100 Token 생성 시간이 다음과 같이 감소했다.

- KV Cache 없음: 약 9.12초
- KV Cache 사용: 약 3.14초

![KV Cache 성능 비교](/assets/img/for_post/2026-08-08-chater2-11.png)

#### The Prefill and Decode Phases

LLM Inference는 크게 **Prefill과 Decode** 두 단계로 나눌 수 있다.

- **Prefill phase**

  사용자가 전달한 전체 Prompt를 한 번에 처리하는 단계이다.

  입력된 모든 Token에 대해 Attention을 계산하기 때문에 **Compute-intensive**한 특성을 가진다.

  Prompt가 길어질수록 처리 비용이 크게 증가한다.

- **Decoding phase**

  Prefill 이후 실제 답변 Token을 **하나씩 생성하는 단계**이다.

  KV Cache를 사용하면 이전 Token의 K/V를 재사용하기 때문에 새로운 Token 중심으로 연산한다.

  Decode는 Token을 반복해서 생성하면서 KV Cache를 계속 읽고 확장하기 때문에 **Memory-intensive**한 특성을 가진다.

![Prefill and Decode](/assets/img/for_post/2026-08-08-chater2-12.png)

![Prefill and Decode 2](/assets/img/for_post/2026-08-08-chater2-13.png)

- **Why Does Learning About the Prefill and Decoding Phases Matter?**

  Prefill과 Decode는 서로 다른 성능 특성을 가지기 때문에 병목도 다르다.

  **긴 Prompt + 짧은 응답**

  - Prefill 비용이 커질 가능성이 높음
  - 예: 긴 PDF 처리

  **짧은 Prompt + 긴 응답**

  - Decode 비용이 커질 가능성이 높음
  - 예: Chatbot, 긴 글 생성

  따라서 LLM Serving을 최적화할 때 어떤 단계가 병목인지 먼저 파악해야 한다.

#### Run the LLM with a Serving Framework

실제 Production 환경에서는 직접 Token Generation Logic을 구현하기보다 **vLLM, SGLang과 같은 Serving Framework**를 사용한다.

Serving Framework는 LLM Inference에 필요한 기능을 제공한다.

- KV Cache 재사용
- Request Scheduling
- Batching / Micro-batching
- 여러 사용자의 동시 요청 처리
- Token Streaming
- Request 취소 및 중단
- 최신 Serving Optimization 적용

Serving Framework를 사용하면 모델별 Inference Logic을 직접 구현하지 않고도 효율적인 LLM 서비스를 구성할 수 있다.

#### Serve the LLM (Qwen) with vLLM

vLLM에서는 `LLM()`으로 모델을 Load하고 `generate()`를 이용해 Inference를 수행할 수 있다.

복잡한 LLM 실행 과정은 vLLM 내부에서 처리된다.

또한 다양한 Configuration을 통해 다음 요소를 조정할 수 있다.

- Memory Management
- KV Cache
- PagedAttention
- Prefix Caching
- Chunked Prefill
- 최대 Sequence 수
- Sampling 방식
- 생성 Token 수

#### Performance Comparison: vLLM Versus Hugging Face Transformers

vLLM은 단순한 사용 편의성뿐만 아니라 LLM Serving 성능을 높이기 위해 설계되었다.

책의 동일 모델과 Prompt를 사용한 예제에서는 다음 결과가 나왔다.

- Hugging Face Transformers: 약 19.58초
- vLLM: 약 1.12초
- 약 17배 속도 향상

Concurrent Request와 Batch 규모가 증가하면 이러한 차이는 더 커질 수 있다.

- **Best Practice: Start Simple, Then Optimize**

  개발 초기에는 Hugging Face Transformers처럼 사용하기 쉬운 도구로 Prototype을 만들 수 있다.

  Production 단계에서는 vLLM과 같은 Serving Framework로 전환하여 다음 요소를 최적화할 수 있다.

  - Latency
  - Throughput
  - Concurrency
  - Resource Utilization

---

## LLM Streaming Serving Basics

일반적인 Generation 방식은 전체 답변 생성이 끝난 후 결과를 반환한다.

Streaming은 전체 생성이 완료될 때까지 기다리지 않고 **생성되는 Token을 즉시 사용자에게 전달하는 방식**이다.

```
Token 생성 → 바로 전달
Token 생성 → 바로 전달
Token 생성 → 바로 전달
```

Chatbot과 같은 서비스에서는 Streaming을 통해 사용자가 첫 응답을 더 빠르게 받을 수 있다.

vLLM에서는 Async Engine을 사용하여 Token을 비동기적으로 반환할 수 있다.

Streaming을 사용하면 생성 중간에 요청을 취소할 수도 있어 불필요한 GPU 연산과 비용을 줄이는 데 도움이 된다.

---

## LLM Batch Serving Basics

하나의 Prompt씩 처리하면 GPU 자원을 충분히 활용하지 못할 수 있다.

Batching은 여러 Input Request를 하나의 Batch로 묶어 **동시에 처리하는 방식**이다.

```
Prompt A ─┐
Prompt B ─┤
Prompt C ─┼→ LLM → 여러 결과
Prompt D ─┘
```

![Batch Serving](/assets/img/for_post/2026-08-08-chater2-14.png)

GPU의 병렬 연산 능력을 활용할 수 있기 때문에 Throughput을 높일 수 있다.

책의 실험에서는 4개의 Prompt를 처리했을 때

- 하나씩 처리: 약 2.39초
- Batch 처리: 약 1.06초

로 약 **2.2배의 Throughput 개선**을 확인했다.

이후에는 요청이 끝날 때마다 새로운 요청을 Batch에 동적으로 추가하는 **Continuous Batching** 같은 고급 최적화 방법도 사용된다.

---

## Summary

LLM은 Decoder-Only Transformer를 기반으로 **Autoregressive 방식으로 Token을 하나씩 생성**한다.

LLM의 기본 처리 흐름은 다음과 같다.

```
Text
↓
Tokenizer / Embedding
↓
Transformer Decoder Blocks
↓
LM Head
↓
Next Token
```

Self-Attention은 Token 간 관계와 Context를 계산하는 핵심 구성 요소이며, LLM Serving에서는 Attention이 많은 Compute와 Memory를 사용한다는 점을 이해하는 것이 중요하다.

LLM Inference는 크게 두 단계로 나뉜다.

- **Prefill**: 전체 Prompt 처리, Compute-intensive
- **Decode**: Token을 하나씩 생성, Memory-intensive

KV Cache를 사용하면 이전 Token의 Key/Value를 재사용하여 Decode 과정의 중복 연산을 줄일 수 있다.

Production 환경에서는 직접 Inference Logic을 구현하기보다 vLLM과 같은 Serving Framework를 사용하며, 이를 통해 KV Cache, Scheduling, Batching, Streaming 등의 기능을 활용할 수 있다.

또한 Streaming은 사용자에게 Token을 빠르게 전달하여 응답성을 높이고, Batching은 여러 요청을 병렬 처리하여 GPU 활용률과 Throughput을 높인다.

이러한 개념들은 이후 LLM Serving의 **성능 병목 분석, 확장성 설계, KV Cache 최적화, Batching 및 Serving Framework 최적화**를 이해하기 위한 기반이 된다.
