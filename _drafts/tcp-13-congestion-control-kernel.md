---
layout: post
title: "혼잡제어: Reno vs Cubic vs BBR 커널 코드 비교"
categories: [TCP]
tags: [뇌절, TCP, 심화, 커널, 혼잡제어]
series: TCP
tier: 심화
order: 13
---

## 선행 지식
**혼잡제어, 큰 그림** - AIMD와 cwnd 기본 개념

## 다룰 내용

- 기본편에서 예고했던 "왜 알고리즘이 여러 종류인가"에 대한 답
- 세 알고리즘의 설계 철학 차이 (loss-based vs model-based)
- 각각의 커널 구현체(`tcp_cubic.c` 등) 핵심 로직 비교
- 왜 리눅스 기본값이 Cubic에서 지금 형태로 왔는지, BBR은 근본적으로 뭐가 다른지

## 이 글이 끝나면
독자는 세 알고리즘을 개념 수준이 아니라 실제 커널 구현 로직 수준에서 비교할 수 있습니다.

## TODO
- [ ] 같은 네트워크 조건에서 세 알고리즘 처리량 비교 실험 설계
