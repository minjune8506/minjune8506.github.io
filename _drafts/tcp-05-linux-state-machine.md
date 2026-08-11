---
layout: post
title: "Linux TCP 상태머신, 소스로 따라가기"
categories: [TCP]
tags: [뇌절, TCP, 커널, Linux]
series: TCP
level: 2
---

## 다룰 내용

- TCP 상태머신 다이어그램을 Linux 커널 소스와 1:1 매칭
- `tcp_v4_connect`, `tcp_rcv_state_process` 등 핵심 함수 코드 리딩
- 상태 전이가 실제 코드에서 어떤 조건문으로 구현돼 있는지
- `ss -tan`으로 실제 상태 관찰하며 코드와 대조

## TODO
- [ ] 커널 버전 명시하고 소스 clone해서 실제 라인 번호로 인용
- [ ] 상태 전이 실험용 최소 재현 스크립트 작성
