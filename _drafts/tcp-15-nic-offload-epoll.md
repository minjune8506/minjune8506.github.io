---
layout: post
title: "NIC 오프로딩(TSO/GRO)과 epoll 내부 구현"
categories: [TCP]
tags: [뇌절, TCP, 심화, OS, 하드웨어]
series: TCP
tier: 심화
order: 15
---

## 선행 지식
**strace로 소켓 syscall 흐름 추적하기** - 심화 단계 마지막 글

## 다룰 내용

- TSO(TCP Segmentation Offload), GRO(Generic Receive Offload)가 뭘 대신 해주는가
- 커널이 어디까지 패킷을 만들고 하드웨어가 어디서부터 이어받는가
- epoll의 내부 자료구조(red-black tree, ready list)와 C10K 문제 해결 원리
- 심화 단계 정리: 여기까지가 "OS/네트워크 엔지니어"가 알면 충분한 레벨

## 이 글이 끝나면
독자는 소프트웨어(커널)와 하드웨어(NIC)의 경계가 어디인지 설명할 수 있습니다.
여기서 심화 단계가 끝나고, 뇌절 단계에서는 직접 손으로 만들고 검증하기 시작합니다.

## TODO
- [ ] `ethtool -k`로 오프로딩 켜고 끄며 성능 차이 측정
