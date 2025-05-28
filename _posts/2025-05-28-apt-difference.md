---
title: "Ubuntu apt 와 apt-get 차이 "
# date: 2025-05-26 23:00:00 +0900
categories: [Tech, Linux]
tags: [ubuntu, apt, apt-get, CLI]
math: true
toc: true
pin: true
# image:
#   path: thumbnail.png
#   alt: image alternative text
---

> CentOS를 사용하다가 Ubuntu를 사용하게 되어서 명령어를 치는데 문득 어디는 apt로 하고 어디는 apt-get으로 해서 궁금해서 찾아보았다


## Ubuntu에서 `man`으로 메뉴얼 검색
```
man apt
```

- DESCRIPTION 확인
    - apt가 apt-get보다 더 상위 레벨의 CLI 패키지 관리 시스템이다.
    - 즉, apt-get 으로 사용할 수 있는 모든 명령어를 apt 로 대체 가능하다.
    - apt-cache , apt-get 이렇게 나누어져 있던 것을 사용 편의를 위해 최신 버전이 나온 것임

```
DESCRIPTION
       apt provides a high-level commandline interface for the package management system. It is intended as an end user interface and enables some options better suited for
       interactive usage by default compared to more specialized APT tools like apt-get(8) and apt-cache(8).

       Much like apt itself, its manpage is intended as an end user interface and as such only mentions the most used commands and options partly to not duplicate information
       in multiple places and partly to avoid overwhelming readers with a cornucopia of options and details.
```

### 궁금하면 `man apt-get` 도 확인해보면 된다! **궁금증 해결 끝 !** 

## References
- [Ubuntu 참조 링크](https://manpages.ubuntu.com/)