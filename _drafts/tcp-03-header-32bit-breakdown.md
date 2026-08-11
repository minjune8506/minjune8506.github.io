---
layout: post
title: "TCP 헤더 32비트 완전분해"
categories: [TCP]
tags: [뇌절, TCP, RFC, 스펙]
series: TCP
level: 1
---

## 다룰 내용

- TCP 헤더 20바이트를 필드 하나하나 비트 단위로 분해
- 예약(Reserved) 비트는 왜 남아있고 지금은 뭘로 쓰이나
- Urgent pointer는 실무에서 실제로 쓰이나 (거의 죽은 필드인 이유)
- 옵션 필드(MSS, Window Scale, SACK, Timestamps)의 존재 이유

## TODO
- [ ] Wireshark로 실제 헤더 캡처해서 필드별 대조
- [ ] 각 옵션 필드가 없던 시절엔 어떤 문제가 있었는지
