---
layout: post
title: "NIC 오프로딩(TSO/GRO)과 epoll 내부 구현"
categories: [TCP]
tags: [뇌절, TCP, OS, 하드웨어]
series: TCP
level: 3
---

## 다룰 내용

- TSO(TCP Segmentation Offload), GRO(Generic Receive Offload)가 뭘 대신 해주는가
- 커널이 어디까지 패킷을 만들고 하드웨어가 어디서부터 이어받는가
- epoll의 내부 자료구조(red-black tree, ready list)와 C10K 문제 해결 원리

## TODO
- [ ] `ethtool -k`로 오프로딩 켜고 끄며 성능 차이 측정
- [ ] epoll 관련 커널 소스 핵심 부분 발췌
