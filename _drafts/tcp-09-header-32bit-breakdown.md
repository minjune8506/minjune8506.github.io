---
layout: post
title: "TCP 헤더 32비트 완전분해"
categories: [TCP]
tags: [뇌절, TCP, 심화, RFC]
series: TCP
tier: 심화
order: 9
---

## 선행 지식
**RFC 793 vs RFC 9293** - 최신 스펙 원문을 기준으로 삼는다는 전제

## 다룰 내용

- TCP 헤더 20바이트를 필드 하나하나 비트 단위로 분해
- 기본편에서 언급했던 개념(시퀀스 넘버, ACK, 윈도우 크기)이 헤더의 정확히 어느 비트인지 매핑
- 예약(Reserved) 비트는 왜 남아있고 지금은 뭘로 쓰이나
- 옵션 필드(MSS, Window Scale, SACK, Timestamps)의 존재 이유

## 이 글이 끝나면
독자는 헤더 덤프를 보고 필드별 값을 직접 읽어낼 수 있습니다.
다음 글(시퀀스 넘버)에서 헤더의 시퀀스 넘버 필드를 더 깊게 파고듭니다.

## TODO
- [ ] Wireshark로 실제 헤더 캡처해서 필드별 대조
