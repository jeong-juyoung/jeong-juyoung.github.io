---
title: Kong Admission Webhook TLS Handshake Error 해결하기
date: 2025-07-24 10:00:00 +0900
categories: [Kubernetes]
tags: [kubernetes, kong, admission-webhook, tls, troubleshooting]
math: false
toc: true
pin: false
---

## 문제 상황

Kong을 AKS 환경에서 운영하던 중 다음과 같은 TLS handshake error가 계속 발생

```yaml
2025/07/23 05:54:05 http: TLS handshake error from 10.244.0.233:44430: EOF
2025/07/23 05:54:05 http: TLS handshake error from 10.244.0.233:44446: EOF
2025/07/23 05:54:05 http: TLS handshake error from 10.244.2.23:32986: EOF
2025/07/23 05:54:05 http: TLS handshake error from 10.244.2.23:32974: EOF
2025/07/23 05:54:05 http: TLS handshake error from 10.244.2.23:33000: remote error: tls: bad certificate
2025/07/23 05:54:05 http: TLS handshake error from 10.244.0.233:44462: EOF
2025/07/23 05:54:05 http: TLS handshake error from 10.244.0.233:44464: EOF
2025/07/23 05:54:05 http: TLS handshake error from 10.244.2.23:33014: EOF
2025/07/23 05:54:05 http: TLS handshake error from 10.244.0.233:44478: EOF
2025/07/23 05:54:05 http: TLS handshake error from 10.244.0.233:44480: EOF
2025/07/23 05:54:05 http: TLS handshake error from 10.244.2.23:33016: EOF
2025/07/23 05:54:06 http: TLS handshake error from 10.244.2.23:33028: EOF
2025/07/23 05:54:06 http: TLS handshake error from 10.244.2.23:33030: EOF
2025-07-23T05:54:15Z info	controller-runtime.certwatcher Updated current TLS certificate {"v": 0}
```

## 원인 분석

### 1. 에러가 발생하는 IP 주소 확인

에러가 발생하는 IP 주소를 확인해보니 `konnectivity-agent` 파드들이었음

```bash
# 10.244.0.233 확인
ubuntu@aks-dev-cluster:~$ kubectl get pods,svc,endpoints -A -o wide | grep 10.244.0.233
kube-system         pod/konnectivity-agent-6c59d5b9b9-xbwqj                               1/1     Running   0          6d8h    10.244.0.233   aks-agentpool-30348648-vmss000000   <none>           <none>

# 10.244.2.23 확인
ubuntu@aks-dev-cluster:~$ kubectl get pods,svc,endpoints -A -o wide | grep 10.244.2.23
kube-system         pod/konnectivity-agent-6c59d5b9b9-h9mtk                               1/1     Running   0          6d8h    10.244.2.23    aks-agentpool-30348648-vmss000001   <none>           <none>
```

`konnectivity-agent`는 AKS에서 Kubernetes API 서버와 노드 간의 터널링을 담당하는 중요한 컴포넌트이다

### 2. Admission Webhook 이해하기

Kubernetes의 **Admission Webhook**은 리소스 생성/수정 시 검증 단계를 담당

```yaml
kubectl apply -f ingress.yaml
         ↓
API Server가 받음
         ↓
Admission Webhook 호출 ← 여기서 Kong이 검증!
         ↓
검증 통과하면 etcd에 저장
```

만약 Admission Webhook이 없다면 어떻게 될까? 궁금해졌다

```yaml
Admission Webhook 없을 때:
1. kubectl apply -f ingress.yaml  
2. API Server가 요청 받음
3. 바로 etcd에 저장 (검증 단계 스킵!)
4. Kong Controller가 변경사항 감지
5. Kong이 실제 설정 적용 시도
   └─ 이때 잘못된 설정이면 Kong 에러 발생!
```

### 3. Webhook 설정 확인

API Server가 Kong을 어떻게 호출하는지 확인

```bash
kubectl get validatingwebhookconfigurations kong-kong-kong-validations -o yaml | grep -A10 clientConfig
```

결과를 보면 443 포트로 호출하고 있는 것 확인

```yaml
clientConfig:
  service:
    name: kong-kong-validation-webhook
    namespace: kong
    port: 443
failurePolicy: Ignore
matchPolicy: Equivalent
name: validations.kong.konghq.com
```

### 4. 포트 불일치 문제 발견

Kong 서비스와 엔드포인트를 확인해보니 포트 불일치가 발견!

**서비스는 443 포트**
```bash
ubuntu@aks-dev-cluster:~/aims-dev-helm/kong$ kubectl get svc -n kong
NAME                           TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                      AGE
kong-kong-validation-webhook   ClusterIP      10.0.204.214   <none>           443/TCP                      48d
```

**실제 엔드포인트는 8080 포트**
```bash
ubuntu@aks-dev-cluster:~/aims-dev-helm/kong$ kubectl get endpoints -n kong
NAME                           ENDPOINTS                               AGE
kong-kong-validation-webhook   10.244.4.148:8080                       48d
```

## 해결 방법

Kong Helm chart의 values.yaml에서 admission webhook의 TLS 설정을 활성화

```yaml
# values.yaml
admissionWebhook:
  enabled: true
  tls:
    enabled: true  # false에서 true로 변경
```

이 설정을 통해
1. Kong admission webhook이 올바른 TLS 인증서를 사용하게 된다.
2. API Server와 Kong 간의 TLS handshake가 정상적으로 이루어진다.
3. 포트 불일치 문제가 해결된다 

## 결론

Kong에서 발생한 TLS handshake error는 admission webhook의 TLS 설정이 비활성화되어 있어서 발생한 문제였다.

`konnectivity-agent`가 API Server를 통해 Kong의 admission webhook을 호출할 때, TLS 인증서 설정이 올바르지 않아 handshake가 실패했던 것이다.

admission webhook의 TLS 설정을 활성화함으로써 이 문제를 해결할 수 있다.

> **참고:** admission webhook 자체를 비활성화하는 방법도 있지만, 이는 Kong 리소스 검증 기능을 잃게 되므로 권장하지 않을듯함 ?