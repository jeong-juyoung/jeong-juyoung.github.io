---
title: "Proxmox VE 설치 및 클러스터 구성하기: 온프레미스 가상화 환경 구축기"
#date: 2025-07-04 10:00:00 +0900
categories: [Tech, Linux]
tags: [proxmox, virtualization, cluster, on-premise, hypervisor]
math: true
toc: true
pin: true
---

> 최근 회사에 사용하지 않는 물리 서버 4대가 있어서, 이를 가상화 환경으로 활용하면 좋겠다는 생각이 들어 Proxmox VE 클러스터를 구축하게 되었다.

## 준비 사항
### 하드웨어 요구사항

- CPU: 64비트 지원, 가상화 기능 활성화 (Intel VT-x/AMD-V)
- RAM: 최소 2GB (권장 8GB 이상)
- Storage: 최소 32GB (권장 100GB 이상)
- Network: 인터넷 연결 및 고정 IP 권장

### 소프트웨어 준비

- Proxmox VE ISO 다운로드
- USB 부팅 도구
    - Windows: Rufus
    - Linux: dd 명령어 또는 Rufus


## 설치 과정
#### 1. 설치 미디어를 넣고 부팅

#### 2. Proxmox VE 설치
- 부팅 옵션 선택
    - Install Proxmox VE (Graphical): 일반적인 GUI 설치 (권장)
    - Install Proxmox VE (Terminal UI): 터미널 기반 설치 (호환성 좋음)
    - Install Proxmox VE (Serial Console): 시리얼 콘솔 환경용


#### 3. 디스크 및 파일시스템 설정
- Proxmox 설치 화면에서 설치할 하드 디스크를 선택하면 됨

    ``` 
    Target Harddisk: /dev/sda (설치할 디스크 선택)
    Filesystem: ext4 (기본값 권장)
    ```
- Option을 선택하면 파티션 설정을 본인이 원하는대로 가능 (아래 계속 참조)


#### 3-1. 파티션 설정 예시
```
hdsize: 100          # 전체 사용할 디스크 크기 (GB)
swapsize: 8          # 스왑 크기 (최소 4GB, 최대 8GB)
maxroot: 80          # OS + 템플릿 + 로그용 (GB)
minfree: 16          # 스냅샷용 여유공간 (GB)
maxvz: 0             # local-lvm 사용 안함 (Ceph 환경에서)
```
- 파티션 설정 설명
    - `hdsize`: 전체 사용할 디스크 크기
    - `swapsize`: 가상 메모리 크기 (물리 메모리와 동일 권장)
    - `maxroot`: OS가 설치되는 파티션 (Proxmox OS, 커널, 패키지)
    - `maxvz`: 데이터 볼륨 최대 크기
    - `minfree`: VG 여유 공간 (128GB 이상 디스크: 기본 16GB)

#### 4. 지역 및 키보드 레이아웃 선택

#### 5. 관리자 계정(root)의 패스워드와 이메일 지정

#### 6. 네트워크 설정
- 환경 설정 예시
```
Hostname : pve-node01-dev
IP Address: 12.3.4.567/24
Gateway: 12.3.4.1
DNS: 168.126.63.1
```

#### 7. 설치 후 초기 설정

- 웹 인터페이스 접속
    - URL: https://서버IP:8006
    - 사용자명: root
    - 패스워드: 설치 시 설정한 패스워드
