---
layout: post
title: "연결 종료 - 4-way handshake와 TIME_WAIT"
categories: [TCP]
tags: [뇌절, TCP, 기본]
series: TCP
tier: 기본
order: 7
---

## 선행 지식
**혼잡제어, 큰 그림** - 기본 단계 마지막 글

## 다룰 내용

- FIN/ACK 4번의 교환이 handshake(3번)보다 한 단계 많은 이유 (half-close)
- 왜 어느 한쪽이든 먼저 종료를 시작할 수 있는가
- TIME_WAIT 상태가 왜 존재하는가, 실무에서 왜 자주 화제가 되는가(포트 고갈 등)
- 기본 단계 정리: 여기까지가 "TCP를 쓰는 사람"이 알면 충분한 레벨

## 이 글이 끝나면
독자는 연결 수립부터 종료까지 전체 생애주기를 처음부터 끝까지 설명할 수 있습니다.
여기서 기본 단계가 끝나고, 심화 단계에서는 "지금까지 배운 걸 스펙 원문과 커널 코드로 검증"하기 시작합니다.

## TODO
- [ ] TIME_WAIT 관련 실무 이슈(SO_REUSEADDR 등) 사례 조사
