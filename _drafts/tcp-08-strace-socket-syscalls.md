---
layout: post
title: "strace로 소켓 syscall 흐름 추적하기"
categories: [TCP]
tags: [뇌절, TCP, OS, syscall]
series: TCP
level: 3
---

## 다룰 내용

- `socket`, `bind`, `listen`, `accept`, `connect`, `epoll_wait` syscall 흐름을 strace로 직접 관찰
- 애플리케이션 레벨 API 호출이 커널로 어떻게 넘어가는지
- 블로킹/논블로킹 소켓의 syscall 레벨 차이

## TODO
- [ ] 간단한 클라이언트/서버 예제 코드 작성 후 strace 캡처
- [ ] 각 syscall의 인자/리턴값을 TCP 상태와 매핑
