---
layout: post
title: "시퀀스 넘버 wraparound과 PAWS"
categories: [TCP]
tags: [뇌절, TCP, RFC, 스펙]
series: TCP
level: 1
---

## 다룰 내용

- 시퀀스 넘버가 왜 32비트인가, 언제 한 바퀴 도는가(wraparound)
- wraparound이 실제로 문제가 되는 상황 (고속 네트워크에서 특히)
- PAWS(Protection Against Wrapped Sequences, RFC 1323)가 이 문제를 어떻게 푸는가
- Timestamps 옵션과의 관계

## TODO
- [ ] wraparound 시나리오를 숫자로 직접 계산해보기
- [ ] PAWS 없이/있이 동작 차이를 실험으로 재현 가능한지 검토
