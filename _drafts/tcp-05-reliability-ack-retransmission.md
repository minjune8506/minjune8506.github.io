---
layout: post
title: "신뢰성 보장 - ACK와 재전송"
categories: [TCP]
tags: [뇌절, TCP, 기본]
series: TCP
tier: 기본
order: 5
---

## 선행 지식
**데이터 전송과 흐름제어** - 윈도우 단위로 데이터가 전송된다는 개념

## 다룰 내용

- ACK가 "누적 확인(cumulative ACK)"이라는 게 무슨 뜻인가
- 패킷이 유실되면 TCP가 그걸 어떻게 알아채는가 (타임아웃, 중복 ACK)
- 재전송이 일어나는 기본 흐름 (RTO 계산의 자세한 공식은 심화편에서)
- Fast retransmit 개념만 가볍게 소개

## 이 글이 끝나면
독자는 "TCP가 왜 신뢰성 있는 프로토콜인가"를 ACK/재전송 메커니즘으로 설명할 수 있습니다.
다음 글(혼잡제어)에서 "유실이 네트워크 혼잡 때문이라면?"으로 이어집니다.

## TODO
- [ ] 중복 ACK 3개 → fast retransmit 예시 시나리오 작성
