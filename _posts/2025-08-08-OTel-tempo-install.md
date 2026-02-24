---
title: OpenTelemetry Operator/Collector와 Grafana Tempo 설치 가이드 (Helm)
date: 2025-08-08 10:00:00 +0900
categories: [DevOps, Kubernetes]
tags: [opentelemetry, tempo, helm, kubernetes, observability, tracing]
math: false
toc: true
pin: false
---

## 개요

애플리케이션의 트레이싱을 빠르게 도입하려면 OpenTelemetry(OTel)로 계측하고, 수집기는 OTel Collector, 백엔드는 Grafana Tempo로 구성하는 패턴이 가장 단순하다고 생각한다 
이 글에서는 다음을 간단히 설명하고, Helm Chart 기반으로 바로 설치해볼 수 있는 가이드를 제공
> 근데 OTel Operator + Alloy + Tempo 조합으로 다시 해볼 것 같음 ^^ 

- OTel Operator: K8s에서 CRD/웹훅으로 auto-instrumentation과 Collector/Target Allocator를 관리
- OTel Collector: 애플리케이션/노드에서 들어오는 텔레메트리를 수집·가공·전송
- Grafana Tempo: 트레이스 백엔드(저장/조회). Metrics Generator로 TraceQL rate() 등 메트릭 기반 쿼리 지원

---

## 핵심 개념 요약

### OpenTelemetry Operator

- K8s Operator로 `Instrumentation` CRD를 통해 애플리케이션 소스 변경 없이 auto-instrumentation 가능
- CRD와 컨트롤러가 연동되어 파드에 에이전트를 주입하고, 지정한 Collector로 데이터를 전송하게 구성
- 웹훅 인증서 옵션(권장 순서) <br>
  1) cert-manager 사용: `admissionWebhooks.certManager.enabled=true` <br>
  2) 사용자 정의 Issuer: `admissionWebhooks.certManager.issuerRef` <br> 
  3) Helm이 자체서명 자동 생성: `admissionWebhooks.autoGenerateCert.enabled=true` <br>
  4) 수동 생성 인증서 파일 제공: `admissionWebhooks.certFile|keyFile|caFile` <br>
  5) 사용자 정의 웹훅/시크릿: `admissionWebhooks.create=false`, `secretName` <br>
  6) 웹훅 비활성화: `admissionWebhooks.create=false` + `ENABLE_WEBHOOKS="false"` 

### OpenTelemetry Collector

- 수집기(Receivers) → 프로세서(Processers) → 내보내기(Exporters) 파이프라인으로 구성
- 배포 방식
  - Deployment: 애플리케이션에서 직접 전송되는 텔레메트리(OTLP 등) 수집에 적합
  - DaemonSet: 노드/시스템 메트릭·로그, kubelet 메트릭 등 호스트 레벨 수집에 적합

### Grafana Tempo

- 단일 바이너리(`tempo`) 또는 마이크로서비스(`tempo-distributed`)로 배포 가능
- 구성 요소(요약)
  - Distributor: 클라이언트 트레이스 수신→해시→Ingester로 라우팅
  - Ingester: 수신 데이터 일괄처리 후 장기 저장
  - Querier / Query-Frontend: 트레이스 조회 처리/병렬화
  - Compactor: 장기 저장소 블록 재구성/수명주기 관리
  - Metrics Generator: Trace → 시계열 메트릭 생성(Prometheus remote-write)
- 주의: TraceQL `rate()` 같은 함수는 Metrics Generator 활성화 및 적절한 프로세서 설정(`service-graphs`, `span-metrics`, `local-blocks`)이 필요

---

## 설치 가이드 (Argo CD + Helm 차트)

### 0) Namespace 및 Helm 저장소 준비

- 사용 차트(Artifact Hub)
  - OpenTelemetry Operator: `https://artifacthub.io/packages/helm/opentelemetry-helm/opentelemetry-operator`
  - OpenTelemetry Collector: `https://artifacthub.io/packages/helm/opentelemetry-helm/opentelemetry-collector`

### 1) cert-manager 설치

- 저는 Helm Chart를 사용하여 ArgoCD에 이미 배포한 상태라서 생략

### 2) OpenTelemetry Operator 설치 (Argo CD)

기본 설정에서 거의 수정 없이 사용했습니다. 인증서는 `cert-manager` 연동만 켰습니다.

`Application` 예시(Argo CD):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: opentelemetry-operator
  namespace: argocd
spec:
  project: default
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  source:
    repoURL: https://open-telemetry.github.io/opentelemetry-helm-charts
    chart: opentelemetry-operator
    targetRevision: 0.0.0 # 차트 버전 고정 권장 (사용 환경에 맞게 지정)
    helm:
      values: |
        admissionWebhooks:
          certManager:
            enabled: true
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

설치 후 CRD 확인

```shell
kubectl get crd | rg opentelemetry | cat
```

### 3) Instrumentation CR 예시(언어별)

아래 예시는 Collector의 OTLP HTTP 엔드포인트로 전송하도록 설정되어 있습니다. 네임스페이스는 `test`입니다.

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: test-java-instrumentation
  namespace: test
spec:
  exporter:
    endpoint: http://opentelemetry-collector.monitoring.svc.cluster.local:4318
  propagators: [tracecontext, baggage, b3]
  sampler:
    type: parentbased_traceidratio
    argument: "1.0"
  java:
    env:
      - name: OTEL_METRICS_EXPORTER
        value: "none"
      - name: OTEL_LOGS_EXPORTER
        value: "none"
      - name: OTEL_EXPORTER_OTLP_PROTOCOL
        value: "http/protobuf"
---
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: test-python-instrumentation
  namespace: test
spec:
  exporter:
    endpoint: http://opentelemetry-collector.monitoring.svc.cluster.local:4318
  propagators: [tracecontext, baggage]
  sampler:
    type: parentbased_traceidratio
    argument: "1.0"
  python:
    env:
      - name: OTEL_METRICS_EXPORTER
        value: "none"
      - name: OTEL_LOGS_EXPORTER
        value: "none"
---
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: test-nodejs-instrumentation
  namespace: test
spec:
  exporter:
    endpoint: http://opentelemetry-collector.monitoring.svc.cluster.local:4318
  propagators: [tracecontext, baggage]
  sampler:
    type: parentbased_traceidratio
    argument: "1.0"
  nodejs:
    env:
      - name: OTEL_METRICS_EXPORTER
        value: "none"
      - name: OTEL_LOGS_EXPORTER
        value: "none"
```

네임스페이스 전체 적용(예)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: test-auto-trace
  annotations:
    instrumentation.opentelemetry.io/inject-java: "test-java-instrumentation"
    instrumentation.opentelemetry.io/inject-python: "test-python-instrumentation"
    instrumentation.opentelemetry.io/inject-nodejs: "test-nodejs-instrumentation"
```

리소스 확인

```shell
kubectl apply -f test-instrumentation.yaml
kubectl get instrumentations.opentelemetry.io -n test | cat

별도 디렉터리로 `Instrumentation`을 관리하여 적용했습니다(예: GitOps).

예시 디렉터리

```text
opentelemetry/
  instrumentations/
    java-instrumentation.yaml
    python-instrumentation.yaml
    nodejs-instrumentation.yaml
    ns-test-auto-trace.yaml
```

Argo CD `Application`으로 디렉터리 전체를 배포(재귀 적용):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: otel-instrumentations
  namespace: argocd
spec:
  project: default
  destination:
    server: https://kubernetes.default.svc
    namespace: test
  source:
    repoURL: https://github.com/<YOUR_ORG>/<YOUR_REPO>
    path: opentelemetry/instrumentations
    targetRevision: main
    directory:
      recurse: true
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```
```

### 4) OpenTelemetry Collector 설치(Deployment)

Collector는 Tempo로 트레이스를 내보내도록 설정합니다. 메타데이터 강화를 위해 `kubernetesAttributes`(및 대응 권한)가 필요합니다.

`values-otel-collector.yaml`

```yaml
mode: deployment

presets:
  kubernetesAttributes:
    enabled: true

clusterRole:
  create: true

config:
  receivers:
    otlp:
      protocols:
        http:
        grpc:
  processors:
    batch: {}
  exporters:
    debug: {}
    otlp/tempo:
      endpoint: tempo.monitoring.svc.cluster.local:4317
      tls:
        insecure: true
  service:
    pipelines:
      traces:
        receivers: [otlp]
        processors: [batch]
        exporters: [debug, otlp/tempo]
```

설치(Argo CD `Application` 예시)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: opentelemetry-collector
  namespace: argocd
spec:
  project: default
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  source:
    repoURL: https://open-telemetry.github.io/opentelemetry-helm-charts
    chart: opentelemetry-collector
    targetRevision: 0.0.0 # 차트 버전 고정 권장 (사용 환경에 맞게 지정)
    helm:
      values: |
        mode: deployment

        presets:
          kubernetesAttributes:
            enabled: true

        clusterRole:
          create: true

        config:
          receivers:
            otlp:
              protocols:
                http:
                grpc:
          processors:
            batch: {}
          exporters:
            debug: {}
            otlp/tempo:
              endpoint: tempo.monitoring.svc.cluster.local:4317
              tls:
                insecure: true
          service:
            pipelines:
              traces:
                receivers: [otlp]
                processors: [batch]
                exporters: [debug, otlp/tempo]
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

노출(선택, 필요 시)

```shell
kubectl -n monitoring get svc opentelemetry-collector -o wide | cat
```

### 5) Grafana Tempo 설치

단일 바이너리 `tempo` 차트를 기준으로 설명합니다. 로컬 저장소를 사용하고, Metrics Generator를 활성화하여 TraceQL `rate()`를 사용할 수 있게 구성합니다.

`values-tempo.yaml`

```yaml
tempo:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318
    jaeger:
      protocols:
        grpc:
          endpoint: 0.0.0.0:14250
        thrift_binary:
          endpoint: 0.0.0.0:6832
        thrift_compact:
          endpoint: 0.0.0.0:6831
        thrift_http:
          endpoint: 0.0.0.0:14268

  storage:
    trace:
      backend: local
      local:
        path: /var/tempo/traces
      wal:
        path: /var/tempo/wal

  metricsGenerator:
    enabled: true

  config:
    server:
      http_listen_port: 3200
    overrides:
      defaults:
        metrics_generator:
          processors:
            - service-graphs
            - span-metrics
            - local-blocks
      per_tenant_override_config: /conf/overrides.yaml
    metrics_generator:
      storage:
        path: "/tmp/tempo"
        remote_write:
          - url: http://kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090/api/v1/write
      traces_storage:
        path: "/tmp/traces"
```

설치(Argo CD `Application` 예시)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: tempo
  namespace: argocd
spec:
  project: default
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  source:
    repoURL: https://grafana.github.io/helm-charts
    chart: tempo
    targetRevision: 0.0.0 # 차트 버전 고정 권장 (사용 환경에 맞게 지정)
    helm:
      values: |
        tempo:
          receivers:
            otlp:
              protocols:
                grpc:
                  endpoint: 0.0.0.0:4317
                http:
                  endpoint: 0.0.0.0:4318
            jaeger:
              protocols:
                grpc:
                  endpoint: 0.0.0.0:14250
                thrift_binary:
                  endpoint: 0.0.0.0:6832
                thrift_compact:
                  endpoint: 0.0.0.0:6831
                thrift_http:
                  endpoint: 0.0.0.0:14268

          storage:
            trace:
              backend: local
              local:
                path: /var/tempo/traces
              wal:
                path: /var/tempo/wal

          metricsGenerator:
            enabled: true

          config:
            server:
              http_listen_port: 3200
            overrides:
              defaults:
                metrics_generator:
                  processors:
                    - service-graphs
                    - span-metrics
                    - local-blocks
              per_tenant_override_config: /conf/overrides.yaml
            metrics_generator:
              storage:
                path: "/tmp/tempo"
                remote_write:
                  - url: http://kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090/api/v1/write
              traces_storage:
                path: "/tmp/traces"
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

서비스 확인 및 Grafana 데이터소스 연결(Tempo HTTP: 3200)

```shell
kubectl -n monitoring get svc | rg tempo | cat
```

### 6) 애플리케이션에 주입(예: Java)

`Deployment`에 annotation만 추가하면 auto-instrumentation이 활성화됩니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app-example
  namespace: test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: java-app-example
  template:
    metadata:
      labels:
        app: java-app-example
      annotations:
        instrumentation.opentelemetry.io/inject-java: "java-instrumentation"
    spec:
      containers:
        - name: app
          image: openjdk:17-jre
          command: ["java", "-jar", "/app/app.jar"]
          ports:
            - containerPort: 8080
```

주입 확인(환경변수/에이전트 경로 등)

```shell
kubectl describe pod <pod-name> -n test | rg -n "JAVA_TOOL_OPTIONS|OTEL_" -n | cat
```

---

## 문제해결 팁(Tempo + TraceQL)

- TraceQL `rate()` 사용 시 500 에러가 난다면 대부분 Metrics Generator가 비활성화이거나 프로세서 설정이 부족한 경우이다.
  - `metricsGenerator.enabled: true`
  - `defaults.metrics_generator.processors: [service-graphs, span-metrics, local-blocks]`


---

## 참고

- Grafana Tempo Helm 차트: `https://github.com/grafana/helm-charts/tree/main/charts/tempo`
- Grafana Tempo Distributed: `https://github.com/grafana/helm-charts/tree/main/charts/tempo-distributed`
- OpenTelemetry Helm 차트: `https://github.com/open-telemetry/opentelemetry-helm-charts`
- 관련 이슈 정리: TraceQL rate() 오류는 Metrics Generator와 `local-blocks` 설정 필요


