---
layout: post
title: "3-way handshake는 왜 이렇게 생겼나"
categories: [TCP]
tags: [뇌절, TCP, 표면]
series: TCP
level: 0
---

## 다룰 내용

- SYN → SYN-ACK → ACK, 이 세 번의 교환이 왜 필요한가 (2-way로는 왜 안 되는가)
- 양쪽이 각자 시퀀스 넘버를 초기화(ISN)해야 하는 이유
- SYN flood 공격이 이 구조의 어떤 약점을 노리는가
- 4-way termination은 왜 handshake보다 한 단계 더 많은가 (half-close)

## TODO
- [ ] 실제 패킷 캡처로 확인
- [ ] "왜 2-way가 아닌가"에 대한 반례/사고실험 정리
