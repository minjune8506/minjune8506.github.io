---
layout: post
title: "Raw socket으로 TCP 패킷 직접 만들기"
categories: [TCP]
tags: [뇌절, TCP, 실습]
series: TCP
tier: 뇌절
order: 17
---

## 선행 지식
**Wireshark로 핸드셰이크 패킷 대조하기** + 심화편 9편(헤더 분해)

## 다룰 내용

- 커널 TCP 스택을 우회해서 raw socket으로 TCP 헤더를 손으로 조립
- 체크섬 계산을 직접 구현 (pseudo header 포함)
- 직접 만든 SYN 패킷을 실제 서버에 보내서 응답 받아보기

## 이 글이 끝나면
독자는 헤더 필드를 "읽을 줄 아는" 수준을 넘어 직접 "만들 줄 아는" 수준이 됩니다.

## TODO
- [ ] 언어 선택 (C? Python + scapy?)
- [ ] 권한/네트워크 환경 제약 사항 정리 (raw socket은 보통 root 필요)
