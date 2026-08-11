---
layout: post
title: "소켓 버퍼·재전송 큐·RTO 계산의 실제 구현"
categories: [TCP]
tags: [뇌절, TCP, 심화, 커널, Linux]
series: TCP
tier: 심화
order: 12
---

## 선행 지식
기본편의 재전송 개념(5편) + **Linux TCP 상태머신**

## 다룰 내용

- `sk_buff` 구조체와 소켓 송수신 버퍼의 관계
- 재전송 큐는 언제 채워지고 언제 비워지는가
- 기본편에서 "타임아웃이 되면 재전송한다"고만 했던 RTO를, 실제 계산식과 커널 코드로 구체화
- Jacobson/Karels 알고리즘 공식이 코드의 어느 줄에 대응하는가

## 이 글이 끝나면
독자는 RTO가 왜 SRTT + 4×RTTVAR로 계산되는지, 그게 커널 어디서 도는지 설명할 수 있습니다.

## TODO
- [ ] SRTT/RTTVAR 관련 커널 변수 이름과 공식 매핑표 작성
