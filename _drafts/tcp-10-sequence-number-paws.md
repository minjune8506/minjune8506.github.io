---
layout: post
title: "시퀀스 넘버 wraparound과 PAWS"
categories: [TCP]
tags: [뇌절, TCP, 심화, RFC]
series: TCP
tier: 심화
order: 10
---

## 선행 지식
**TCP 헤더 32비트 완전분해** - 시퀀스 넘버가 32비트 필드라는 것

## 다룰 내용

- 시퀀스 넘버가 32비트인 이유, 언제 한 바퀴 도는가(wraparound)
- wraparound이 실제로 문제가 되는 상황 (고속 네트워크에서 특히)
- PAWS(Protection Against Wrapped Sequences, RFC 1323)가 이 문제를 어떻게 푸는가
- 기본편의 "재전송" 개념과 wraparound이 충돌할 수 있는 지점

## 이 글이 끝나면
독자는 왜 초고속 네트워크에서 32비트 시퀀스 넘버가 한계가 되는지, PAWS가 그걸 어떻게 보완하는지 설명할 수 있습니다.
다음 글부터는 스펙을 떠나 실제 커널 구현으로 넘어갑니다.

## TODO
- [ ] wraparound 시나리오를 숫자로 직접 계산해보기
