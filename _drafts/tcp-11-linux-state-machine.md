---
layout: post
title: "Linux TCP 상태머신, 소스로 따라가기"
categories: [TCP]
tags: [뇌절, TCP, 심화, 커널, Linux]
series: TCP
tier: 심화
order: 11
---

## 선행 지식
기본편의 연결 수립/종료(3, 7편) - 상태 다이어그램(SYN_SENT, ESTABLISHED, TIME_WAIT, CLOSING 등)에 대한 이해

## 다룰 내용

- 기본편에서 그림으로만 봤던 상태머신(7편에서 추가한 CLOSING 포함)을 Linux 커널 소스와 1:1 매칭
- `tcp_v4_connect`, `tcp_rcv_state_process` 등 핵심 함수 코드 리딩
- 상태 전이가 실제 코드에서 어떤 조건문으로 구현돼 있는지
- **SYN cookie**: 3편에서 예고했던 SYN flood 방어 기법. backlog 큐 없이 SYN-ACK의 시퀀스 넘버 자체에 정보를 암호화해서 담아, ACK가 돌아왔을 때만 상태를 만드는 방식 — 커널 코드(`cookie_v4_init_sequence` 등)로 확인
- `ss -tan`으로 실제 상태 관찰하며 코드와 대조

## 이 글이 끝나면
독자는 "이론상의 상태머신"이 아니라 실제 커널이 도는 코드 레벨에서 상태 전이를 확인할 수 있고, 3편에서 미뤄뒀던 SYN cookie의 작동 원리도 설명할 수 있습니다.

## TODO
- [ ] 커널 버전 명시하고 소스 clone해서 실제 라인 번호로 인용
