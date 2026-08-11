---
layout: post
title: "Wireshark로 핸드셰이크 패킷 뜯어서 RFC와 대조하기"
categories: [TCP]
tags: [뇌절, TCP, 실습]
series: TCP
tier: 뇌절
order: 16
---

## 선행 지식
기본편 3편(3-way handshake) + 심화편 8~9편(RFC, 헤더 분해)

## 다룰 내용

- 실제 handshake를 캡처해서 필드 값을 RFC 9293 스펙과 하나씩 대조
- ISN이 실제로 랜덤하게 생성되는지 여러 번 캡처해서 확인
- Window scaling, SACK 옵션이 실제 협상되는 과정 관찰
- "글로만 배운 것"을 눈으로 직접 확인하는 첫 번째 뇌절 실습

## 이 글이 끝나면
독자는 지금까지 텍스트로만 배운 handshake를 실제 패킷 바이트로 직접 확인한 상태가 됩니다.

## TODO
- [ ] 캡처 스크린샷/pcap 파일 첨부
