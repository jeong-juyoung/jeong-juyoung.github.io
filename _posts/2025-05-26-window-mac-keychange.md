---
title: "Windows11 에서 맥북 키배열로 변경"
# date: 2025-05-26 23:00:00 +0900
categories: [Tech, Windows]
tags: [Windows11, 맥북, 키보드, 키배열, 단축키, PowerToys, 키맵핑]
math: true
toc: true
pin: true
# image:
#   path: thumbnail.png
#   alt: image alternative text
---

> 바꾸려는 이유는 집에선 맥북 , 회사에선 윈도우로 사용중 헷갈려서 나한테 더 익숙한 맥북 키배열로 바꾸려고 한다 <br>
나중에 안찾아보려고 작성하는 글

# 맥처럼 키배열 바꾸기 
> 아래 두 가지를 변경 
- `한/영`키와 `CAPS LOCK`키 위치 변경
- `Ctrl`키와 `Alt`키 위치 변경 

### 1. 레지스트리 편집기 실행 
![Desktop View](/assets/img/for_post/2025-05-25-window-1.png){: .normal }

### 2. Keyboard Layouts으로 이동
- 아래 순서대로 이동 
    - HKEY_LOCAL_MACHINE > SYSTEM > CurrentControlSet > Control > Keyboard Layout
![Desktop View](/assets/img/for_post/2025-05-25-window-2.png){: .normal }


### 3. 이진값 생성
- 우클릭 > 새로 만들기 > 이진값 선택
![Desktop View](/assets/img/for_post/2025-05-25-window-3.png){: .normal }

### 4. 이진 데이터 수정
- 복붙 불가능 직접 입력 필요
- 이름을 Scancode Map으로 변경 후, 우클릭해서 `이진 데이터 수정`을 선택 
    ```
    00 00 00 00 00 00 00 00
    02 00 00 00 72 00 3A 00
    3A 00 38 E0 1D 00 38 00
    38 00 1D 00 00 00 00 00
    ```
![Desktop View](/assets/img/for_post/2025-05-25-window-4.png){: .normal }

### 5. 시스템 재부팅 
- 모든 설정이 끝났다면 컴퓨터 재부팅을 한다. 재시작이 완료되었다면 '한/영'키와 'Caps Lock'키가 'Ctrl'키와 'Alt'키가 바뀐 것을 확인할 수 있다.


## References
- https://caffeineoverflow.tistory.com/23