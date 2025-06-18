---
title: "Azure AKS에서 ArgoCD cert-manager-webhook OutofSync Loop 해결기"
#date: 2025-05-26 23:00:00 +0900
categories: [DevOps, Kubernetes]
tags: [argocd, cert-manager, kubernetes, aks, azure, webhook, sync, gitops]
math: true
toc: true
pin: true
# image:
#   path: thumbnail.png
#   alt: image alternative text
---

> 3년간 온프레미스 Kubernetes 환경에서만 ArgoCD를 운영해왔던 나에게, Azure AKS 환경은 새로운 도전이었다. "어차피 Kubernetes니까 비슷하겠지"라는 안일한 생각으로 기존 ArgoCD 설정을 그대로 AKS에 적용했다가... 에러 발견 !

## 문제 상황
- ArgoCD에서 cert-manager-webhook OutofSync Loop 현상 발생 
    - cert-manager-webhook의 ValidatingWebhookConfiguration이 지속적으로 OutofSync 상태 

## 원인 분석
- 이 문제의 주요 원인은 AKS/Azure가 특정 필드를 자동으로 수정하기 때문

- **control-plane** 관련 `namespaceSelector`
- `kubernetes.azure.com/managedby` 관련 필드 
- Azure는 관리형 Kubernetes 서비스의 특성상 이러한 필드들을 자동으로 추가하거나 수정하는데, 이것이 ArgoCD의 GitOps 방식과 충돌하게 됨

## 해결 방법
1. `Ignore Differences` 설정
    ```yaml
    # argocd-cm ConfigMap에 추가
    resource.customizations.ignoreDifferences.admissionregistration.k8s.io_ValidatingWebhookConfiguration: |
    jqPathExpressions:
        - '.webhooks[].namespaceSelector.matchExpressions[] | select(.key == "control-plane")'
        - '.webhooks[].namespaceSelector.matchExpressions[] | select(.key == "kubernetes.azure.com/managedby")'
    ```

2. ArgoCD 컴포넌트 재시작



### References
- [ArgoCD Issue #4276](https://github.com/argoproj/argo-cd/issues/4276#issuecomment-907797060)
- [cert-manager Issue #4114](https://github.com/cert-manager/cert-manager/issues/4114#issuecomment-1008162907)
