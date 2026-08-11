---
layout: post
title: "소켓 버퍼·재전송 큐·RTO 계산의 실제 구현"
categories: [TCP]
tags: [뇌절, TCP, 커널, Linux]
series: TCP
level: 2
---

## 다룰 내용

- `sk_buff` 구조체와 소켓 송수신 버퍼의 관계
- 재전송 큐는 언제 채워지고 언제 비워지는가
- RTO(Retransmission Timeout) 계산이 커널 코드에서 실제로 어떻게 도는지
- Jacobson/Karels 알고리즘 공식이 코드의 어느 줄에 대응하는가

## TODO
- [ ] SRTT/RTTVAR 관련 커널 변수 이름과 공식 매핑표 작성
- [ ] 재전송이 실제로 일어나는 상황을 tcpdump로 재현
