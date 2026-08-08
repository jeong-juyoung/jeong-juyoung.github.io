---
layout: post
title: "LLM Serving and Optimization - Chapter 1. Introduction to Model Serving and Optimization"
date: 2026-08-08 10:00:00 +0900
categories: [dev, llm]
tags: [LLM, serving, optimization, vLLM, model-serving, inference]
---

*Hands-On LLM Serving and Optimization* 을 읽으며 정리한 내용이다.

LLM을 제대로 공부해본 적이 없었는데, 모르는 개념이 많아서 쉽진 않지만, 그래도 읽은 만큼 정리해두려고 한다.

---

## Why LLM Serving and Optimization?

**LLM 서비스 제공 및 최적화를 선택하는 이유는 무엇일까요?**

과거 AI는 연구실에서만 사용하는 모델이 많았다.  
현재는 실시간 서비스로 활용되고 있다.

- ChatGPT
- Netflix 추천
- Amazon 추천
- 은행 사기 탐지
- AI 챗봇

이제 중요한 것은 **모델을 잘 학습시키는 것**뿐만 아니라 **실제 서비스에서 안정적으로 실행하는 것**이다.

---

## Chapter 1. Introduction to Model Serving and Optimization

### Anatomy of a Model

![Anatomy of a Model](/assets/img/for_post/2026-08-08-chater1-1.png)

**모델의 구성 요소**

- 모델 데이터
- 모델 아키텍처
- 모델 실행 코드

---

### Model Lifecycle: From Training to Serving

**모델 생명주기: 학습에서 서빙까지**

- **Data Collection**
  - 사용자 데이터, 로그, 센서 데이터 등을 수집하고 학습 가능한 형태로 정제한다.
- **Training and Fine-tuning**
  - 데이터를 이용하여 모델을 학습한다.
  - LLM은 대규모 사전학습(Pretraining) 이후 특정 목적에 맞게 Fine-tuning을 수행하는 경우가 많다.
- **Evaluation**
  - Accuracy, Loss 등 다양한 지표를 이용하여 모델 성능을 평가하고 실제 서비스에 사용할 수 있는지 검증한다.
- **Deployment**
  - 학습된 모델을 Production 환경에서 사용할 수 있도록 패키징하고 배포한다.
- **Serving**
  - 배포된 모델을 API나 서비스 형태로 제공하여 새로운 입력 데이터에 대한 Inference를 수행한다.
- **Optimization and Iteration**
  - 서비스 운영 중 성능, 안정성, 비용을 지속적으로 개선하며, 필요하면 다시 학습을 수행하여 모델을 업데이트한다.
  - Training과 Serving은 서로 다른 단계이며, 모델의 가치는 실제 서비스에 배포되어 운영될 때 비로소 만들어진다.

---

### What Is Model Serving?

비즈니스 요구사항에 따라 다양한 환경에서 Serving할 수 있다.

- **On-device**: 디바이스 내부에서 모델 실행
- **On-premises**: 회사 내부 인프라에서 모델 실행
- **On-cloud**: 클라우드 환경에서 모델 실행

---

### Why Study Model Serving?

Cloud Vendor(AWS, Azure 등)의 Model Serving 서비스를 사용하는 것만으로도 서비스를 운영할 수 있다.  
하지만 실제 서비스에서는 단순히 서비스를 사용하는 것만으로는 충분하지 않다.

- 다양한 Serving 옵션 중 요구사항에 맞는 구성을 선택해야 한다.
- 기존 시스템과의 연동을 고려해야 한다.
- 비용, 성능, 확장성을 고려한 최적화가 필요하다.

따라서 Model Serving의 동작 원리를 이해해야 Cloud Vendor의 서비스를 적절하게 선택하고 활용할 수 있다.

---

### Why Optimize Model Serving (Especially for LLMs)?

모델을 배포해서 정상 작동한다고 '끝'이 아니다. LLM은 특히 별도 최적화 없이 서빙하면 **실사용자가 몰릴 때 빠르게 문제**가 생긴다.

- 부하가 걸리면 지연시간(latency) 증가
- 처리량(throughput)이 하드웨어 성능보다 훨씬 낮은 수준에서 정체
- 사용량이 늘수록 비용이 선형적으로(또는 그 이상) 증가

데모나 파일럿 단계에선 잘 작동하던 시스템이, **실제 유저가 유입**되면 경제적/운영적으로 감당 불가능해지는 경우가 많다.

**핵심 개념 정의**

- **모델 서빙 최적화**: 지연시간 감소, 처리량 증가, 리소스 활용도 개선 등을 통해 서빙 성능을 향상시키는 작업
  - 낮은 latency
  - 높은 throughput
  - 높은 GPU 활용률
  - 낮은 cost per request
  - 안정적인 tail latency
  - 효율적인 메모리 사용
- **목표**: 비용을 통제하면서 서빙 효율을 극대화하는 것

**왜 특히 LLM에서 중요한가**

- LLM은 연산 요구량이 매우 커서 **운영 비용 부담이 큼**
- 예시: Alphabet 회장 John Hennessy는 2023년 로이터와의 인터뷰에서 "LLM 요청 1건 처리 비용이 전통적인 키워드 검색보다 10배 비쌀 수 있고, 이는 수십억 달러 규모의 추가 비용으로 이어질 수 있다"고 언급
- **LLM에서는 최적화**가 "선택"이 아니라 **"필수"**에 가까움

#### Example: Using a Model Serving Framework (vLLM) to Improve LLM Throughput

**vLLM이란?**

**LLM을 빠르게 실행하기 위한 Serving Framework**

- **LLM** → AI 모델
- **Serving Framework (vLLM, Triton, SGLang)** → AI 모델을 실행하는 프로그램

| 구분 | Model Training | Model Serving |
| --- | --- | --- |
| 목적 | 모델 학습 | Inference 수행 |
| 연산 | Forward + Backpropagation + Weight 업데이트 | Forward만 수행 |
| 최적화 | 높은 학습 성능(Throughput) | 낮은 지연시간(Latency)과 높은 처리량(Throughput) |
| 자원 | 대규모 GPU/TPU 기반 분산 학습 | CPU, GPU, Inference 전용 가속기 |

---

## Model Serving Paradigms

Model Serving에는 하나의 정답이 있는 것이 아니라, 서비스 요구사항에 따라 여러 방식이 사용된다.  
각 방식은 Latency, Cost, Scalability, 운영 복잡도 측면에서 서로 다른 장단점을 가진다.

### On-Device (Edge) Serving

On-device Serving은 모델을 스마트폰, 드론, 카메라, 로봇 등에서 직접 실행하는 방식이다.

![On-Device Serving](/assets/img/for_post/2026-08-08-chater-2.png)

**주요 장점**

- **Cost-effective**: 클라우드 호출 비용을 줄일 수 있음
- **Efficient**: 네트워크 지연 없이 빠른 응답 가능
- **Private**: 데이터가 기기 밖으로 나가지 않아 개인정보 보호에 유리
- **Personalized**: 사용자 데이터를 기반으로 기기 내에서 개인화 가능

#### On-device serving design

**Model Runtime (모델 런타임)**

다양한 하드웨어와 OS에서 모델을 효율적으로 실행하도록 도와주는 소프트웨어 계층이다.

- LiteRT
- ONNX Runtime
- Core ML

**Model Wrapper (모델 래퍼)**

애플리케이션이 모델을 쉽게 호출할 수 있도록 모델 실행 과정을 감싸는 컴포넌트이다.

- 입력 데이터 전처리
- 모델 로딩
- 모델 실행
- 출력 후처리

**전체 흐름**

```
Application Logic
→ Model Wrapper
→ Model Runtime
→ Local Hardware에서 Inference
```

#### Preparing a model for on-device serving

학습된 모델을 디바이스에서 실행하려면 다음 과정이 필요하다.

1. 학습 포맷을 Runtime이 사용하는 포맷으로 변환
2. 변환 후 정확도 검증
3. 디바이스에서 성능 측정
4. 앱에 포함하여 배포

Qualcomm AI Hub 같은 도구를 사용하면 이러한 과정을 자동화할 수 있다.

#### When on-device serving is the right choice

다음과 같은 환경에서 적합하다.

- 개인정보가 기기 밖으로 나가면 안 되는 경우
- 밀리초 단위의 매우 낮은 Latency가 필요한 경우
- 네트워크 연결이 불안정한 경우
- IoT, 로봇, 스마트시티 등 제한된 환경에서 지속적인 Inference가 필요한 경우

다음과 같은 제약도 있다.

- CPU, GPU, Memory 등 연산 자원이 제한적
- 대형 모델 실행이 어려움
- 전력 소비가 큼
- 모델 업데이트가 어려움
- 디바이스마다 하드웨어 지원이 다름

이러한 제약이 커지면 On-Premises 또는 Cloud Serving으로 이동하게 된다.

---

### Single-Model Service

하나의 모델과 하나의 모델 버전을 독립적인 웹 서비스로 배포하는 방식이다.  
각 서비스는 HTTP 또는 gRPC API를 통해 Prediction 요청을 받는다.

구조는 일반적인 Microservice Architecture와 유사하다.

```
Client
→ API
→ Model Serving Container
→ Inference
```

![Single-Model Service](/assets/img/for_post/2026-08-08-chater1-3.png)

#### Containerization Is the Foundation of Modern Model Serving

서버 기반 Model Serving에서는 대부분 모델을 Container 내부에서 실행한다.

Container에는 일반적으로 다음 요소가 포함된다.

- 모델 관리 로직
- 모델 실행 로직
- 필요한 Library와 Dependency
- Runtime 설정

Docker Container나 Kubernetes Pod 형태로 배포하고, HTTP 또는 gRPC API로 모델 기능을 제공한다.

Single-Model Serving Container는 크게 세 가지 요소로 구성된다.

- **API-Server**: 외부 애플리케이션이 HTTP 또는 gRPC를 통해 Inference 요청을 보낼 수 있도록 한다.
- **Model Management**: 모델을 다운로드하고 로컬에 저장한 뒤 Inference Backend에 로딩한다. 새로운 모델 버전이 배포되면 이를 감지해 갱신할 수도 있다.
- **Inference Backend**: 실제로 모델을 실행하는 부분이다.
  - TensorFlow Serving
  - TorchServe
  - vLLM
  - TensorRT-LLM

#### Routing choices in single-model serving

단순 Round-Robin 방식은 요청마다 처리 시간이 다르기 때문에 비효율적일 수 있다.

예를 들어 LLM에서 5,000 Token 요청과 100 Token 요청은 처리 시간이 크게 다르다.

따라서 다음과 같은 방식이 사용될 수 있다.

- Weighted Round Robin
- Least Connections
- Least Response Time
- Dynamic Load Balancing

Dynamic Load Balancing은 CPU, GPU, Memory, Queue Length 등의 실시간 상태를 기준으로 요청을 분배한다.

#### Horizontal and vertical scaling

- **Horizontal Scaling**
  - 트래픽이 증가하면 동일한 Serving Container를 추가하는 방식이다.
  - Kubernetes에서는 HPA를 이용해 CPU, Memory, Latency, 요청 수 등을 기준으로 Pod 수를 자동 조절할 수 있다.
- **Vertical Scaling**
  - 모델이 너무 커서 하나의 GPU 또는 Machine에 들어가지 않을 경우 더 강력한 GPU를 사용하거나 여러 GPU에 모델을 분산하는 방식이다.
  - vLLM에서는 `tensor-parallel-size` 같은 설정을 통해 여러 GPU에 모델을 분산할 수 있다.

#### Prioritizing Intra-Node Serving over Inter-Node Serving

가능하다면 모델을 여러 Machine에 분산하기보다 하나의 Machine에 여러 GPU를 사용해 실행하는 것이 좋다.

여러 Machine에 분산하면 Network Overhead와 Synchronization 비용이 발생하여 Latency가 증가하고 운영도 복잡해지기 때문이다.

#### Why single-model service is the default starting point

Single-Model Service는 단순하고 범용적이기 때문에 기본적인 Serving 방식으로 사용된다.

**장점**

- 모델 간 Resource 경쟁이 없어 성능이 좋음
- 모델별 독립적인 Scaling 가능
- 배포와 Debugging이 쉬움
- 장애가 다른 모델에 영향을 주지 않음
- 모델별 Hardware 최적화 가능

하지만 모델 수가 많아지면 서비스 수가 함께 증가하기 때문에 운영 비용과 Resource 낭비가 커질 수 있다.  
이러한 문제를 해결하기 위해 Multi-Model Service가 사용된다.

---

### Multi-Model Service

Multi-Model Service에서는 다음 두 요소가 중요하다.

- **Model Server Inference Backend**: 여러 종류의 모델을 하나의 통합된 API로 실행한다. 내부적으로 TensorFlow, ONNX, PyTorch 등 여러 Backend를 지원할 수 있다. 대표적인 예가 NVIDIA Triton Inference Server이다.
- **Model Cache Management**: 필요한 모델을 Storage에서 가져와 Load하고, Resource가 부족할 경우 사용 빈도가 낮은 모델을 Unload한다. 주로 LRU(Least Recently Used) 방식으로 관리한다.

![Multi-Model Service](/assets/img/for_post/2026-08-08-chater1-4.png)

**Triton Inference Server: A Unified, Multi-Model Serving Platform**

Triton Inference Server는 여러 Model Format과 Framework를 하나의 API로 서비스할 수 있는 오픈소스 Serving Solution이다.

여러 모델을 동시에 관리할 수 있으며, GPU와 Memory 범위 내에서 다양한 모델을 동적으로 Load하고 실행할 수 있다.

요청 처리 흐름은 다음과 같다.

- 모델이 이미 Load되어 있으면 바로 실행
- Load되어 있지 않으면 Storage에서 가져와 Load 후 실행
- Memory 사용량이 높으면 사용 빈도가 낮은 모델을 Unload

#### Routing and autoscaling in multi-model service

Multi-Model Service에서는 두 가지 문제가 발생한다.

- **Routing**: 요청한 모델이 이미 Load되어 있는 Container로 요청을 보내야 한다. 그렇지 않으면 Model Loading으로 인해 Cold Start가 발생할 수 있다.
- **Per-model Scaling**: 모델마다 Traffic이 다르기 때문에 많이 사용되는 모델은 더 많은 Replica가 필요하다.

이를 해결하기 위해 Routing Layer에서 다음을 관리한다.

- 모델별 Replica 수
- 모델이 어느 Container에 위치하는지에 대한 Route Map

#### When multi-model service becomes necessary

Multi-Model Service는 많은 모델을 적은 Resource로 운영해야 할 때 비용 효율적이다.

하지만 다음 상황에서는 적합하지 않을 수 있다.

- 모델이 너무 커서 GPU Resource 공유가 어려운 경우
- 모델 Traffic이 많아 항상 Memory에 Load되어 있어야 하는 경우
- 모델별 Security Policy가 다른 경우
- Cache, Routing, Scaling, Dependency 관리 등 운영 복잡도가 너무 높은 경우

Single-Model Service와 Multi-Model Service는 경쟁 관계가 아니라 서로 보완적인 방식이다.

---

### Model Serving Platforms

서비스 규모가 커지면 Single-Model과 Multi-Model Service만으로는 운영하기 어려워진다.

**주요 문제**

- 하나의 요청을 처리하기 위해 여러 모델이 함께 동작해야 함
- 여러 AI Application 사이에서 GPU, CPU, Memory를 효율적으로 배분해야 함

Model Serving Platform은 이를 통합적으로 관리한다.

![Model Serving Platforms](/assets/img/for_post/2026-08-08-chater1-5.png)

**주요 구성 요소**

- **Resource Groups (리소스 그룹)**: Application 또는 Business Scenario별로 CPU, GPU, Memory를 분리하여 할당한다. 이를 통해 Resource 경쟁을 줄이고 비용과 성능을 관리한다.
- **Graph Execution (그래프 실행)**: 여러 모델이 순차적 또는 병렬적으로 실행되는 Inference Workflow를 관리한다.

예를 들어 하나의 AI 요청이 다음과 같은 흐름을 가질 수 있다.

```
Intent Classification
→ Embedding
→ Retrieval
→ LLM
→ Safety Filter
```

---

## Summary

모델은 단순한 정적 파일이 아니라 다음 세 가지 요소로 구성된 실행 가능한 구성요소이다.

- Model Data
- Model Architecture
- Model Execution Code

Model Serving은 학습된 모델을 Production 환경에서 실행하여 새로운 입력에 대해 Inference를 제공하는 과정이다.

Serving에서는 Scalability, Latency, Monitoring, Security, Cost 등을 함께 고려해야 한다.

특히 LLM은 Inference 비용이 높기 때문에 Serving Optimization이 중요하며, Serving Framework와 최적화 기법을 통해 같은 Hardware에서도 Throughput과 Latency를 크게 개선할 수 있다.

대표적인 Model Serving Paradigm은 다음과 같이 발전한다.

```
On-Device
    ↓
Single-Model Service
    ↓
Multi-Model Service
    ↓
Model Serving Platform
```

각 방식은 요구되는 Latency, 비용, Scalability, 운영 복잡도에 따라 선택한다.

Model Serving은 모델 학습 알고리즘을 만드는 영역보다는 **모델을 실제 Application에 안정적이고 효율적으로 배포하고 운영하는 Engineering 영역**에 가깝다.
