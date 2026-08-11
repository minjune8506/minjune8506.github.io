---
layout: post
title: "혼잡제어: Reno vs Cubic vs BBR 커널 코드 비교"
categories: [TCP]
tags: [뇌절, TCP, 커널, 혼잡제어]
series: TCP
level: 2
---

## 다룰 내용

- 세 알고리즘의 설계 철학 차이 (loss-based vs model-based)
- 각각의 커널 구현체(`tcp_cubic.c` 등) 핵심 로직 비교
- 왜 리눅스 기본값이 Cubic에서 지금 형태로 왔는지
- BBR이 기존 방식과 근본적으로 다른 지점

## TODO
- [ ] 같은 네트워크 조건에서 세 알고리즘 처리량 비교 실험 설계
- [ ] `sysctl`로 알고리즘 전환하며 실측
