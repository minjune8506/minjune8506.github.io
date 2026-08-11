---
layout: post
title: "strace로 소켓 syscall 흐름 추적하기"
categories: [TCP]
tags: [뇌절, TCP, 심화, OS, syscall]
series: TCP
tier: 심화
order: 14
---

## 선행 지식
기본편 전체 - 애플리케이션 관점에서 본 연결 생애주기

## 다룰 내용

- `socket`, `bind`, `listen`, `accept`, `connect`, `epoll_wait` syscall 흐름을 strace로 직접 관찰
- 지금까지 "커널이 알아서 한다"고 넘어갔던 부분을 애플리케이션-커널 경계에서 확인
- 블로킹/논블로킹 소켓의 syscall 레벨 차이

## 이 글이 끝나면
독자는 애플리케이션 코드 한 줄이 어떤 syscall로 이어지고, 그게 지금까지 배운 TCP 상태와 어떻게 맞물리는지 추적할 수 있습니다.

## TODO
- [ ] 간단한 클라이언트/서버 예제 코드 작성 후 strace 캡처
