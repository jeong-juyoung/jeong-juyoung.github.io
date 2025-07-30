---
title: Grafana Tempo 트레이스 조회 오류 해결하기 - Metrics Generator 설정
date: 2025-07-30 10:00:00 +0900
categories: [Kubernetes]
tags: [grafana, tempo, traceql, metrics-generator, observability, trace]
math: false
toc: true
pin: false
---

## 문제 상황

### TraceQL rate() 함수 오류

TraceQL의 `rate()` 함수를 사용하여 메트릭을 조회하려고 시도했지만 다음과 같은 오류가 발생

```
failed to execute traceql query rate status 500 internal server error error finding generator in querier.queryRangeRecent empty ring
```
![TraceQL Rate 함수 오류](/assets/img/for_post/2025-07-309-traceql-error.png){: .normal }<br>

## 원인 분석

### 1. TraceQL rate() 함수의 동작 원리

TraceQL의 `rate()` 함수는 단순히 trace 데이터를 조회하는 것이 아닌, **미리 계산된 메트릭 데이터**를 필요로 하다고 한다 
이 메트릭 데이터는 Tempo의 **metrics generator** 컴포넌트에 의해 생성된다

```yaml
TraceQL rate() 함수 실행 과정
1. Grafana에서 rate() 함수 실행
2. Tempo가 metrics generator에서 생성된 메트릭 조회 시도
3. metrics generator가 비활성화되어 있으면 "empty ring" 오류 발생
```

### 2. Metrics Generator의 역할

Tempo의 metrics generator는 들어오는 trace 데이터를 실시간으로 분석하여 다음과 같은 메트릭을 생성

- **Request Rate**: 서비스별 요청 수
- **Error Rate**: 서비스별 에러율
- **Duration**: 서비스별 응답 시간
- **Service Dependencies**: 서비스 간 호출 관계

### 3. 오류 메시지 분석

`error finding generator in querier.queryRangeRecent empty ring` 오류는 다음을 의미

- **querier**: Tempo의 쿼리 처리 컴포넌트
- **generator**: metrics generator 컴포넌트를 찾을 수 없음
- **empty ring**: metrics generator 인스턴스가 등록되지 않음

## 해결 방법

### 1. Metrics Generator 활성화

Tempo 설정에서 metrics generator를 활성화해야 한다

```yaml
# tempo.yaml 또는 values.yaml
tempo:
  metricsGenerator:
    enabled: true  # false에서 true로 변경
```

### 2. Metrics Generator 프로세서 설정

metrics generator가 어떤 종류의 메트릭을 생성할지 프로세서를 설정

```yaml
tempo:
  config:
    defaults:
      metrics_generator:
        processors:
          - service-graphs    # 서비스 간 관계와 호출 패턴 추적
          - span-metrics      # 개별 span의 지속시간, 에러율 등 메트릭 생성
          - local-blocks      # 로컬 블록 처리를 위한 메트릭
```

### 3. 각 프로세서의 역할

**service-graphs**
- 서비스 간의 의존성 그래프 생성
- 호출 패턴과 관계 추적
- 서비스 맵 데이터 제공

**span-metrics**
- 개별 span으로부터 메트릭 추출
- 지속시간, 에러율, 처리량 계산
- rate() 함수가 주로 사용하는 메트릭

**local-blocks**
- 로컬 블록 처리 최적화
- 메트릭 생성 성능 향상


## 결론

### 1. TraceQL 함수 관련 오류
- **`rate()` 함수 사용 시**: Tempo의 **metrics generator가 반드시 활성화**되어야 함
- metrics generator는 trace 데이터로부터 실시간으로 메트릭을 생성하는 핵심 컴포넌트
- 적절한 프로세서 설정(`service-graphs`, `span-metrics`, `local-blocks`)이 필요

