---
layout: post
title: "Wireshark로 핸드셰이크 패킷 뜯어서 RFC와 대조하기"
categories: [TCP]
tags: [뇌절, TCP, 실습]
series: TCP
level: 4
---

## 다룰 내용

- 실제 handshake를 캡처해서 필드 값을 RFC 9293 스펙과 하나씩 대조
- ISN이 실제로 랜덤하게 생성되는지 여러 번 캡처해서 확인
- Window scaling, SACK 옵션이 실제 협상되는 과정 관찰

## TODO
- [ ] 캡처 스크린샷/pcap 파일 첨부
- [ ] 3편(헤더 분해) 글과 상호 링크
