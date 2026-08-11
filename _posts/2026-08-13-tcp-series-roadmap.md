---
layout: post
title: "TCP 뇌절 시리즈: 로드맵"
date: 2026-08-13 10:00:00 +0900
categories: [TCP]
tags: [뇌절, 로드맵, TCP]
---

**뇌절**의 첫 시리즈 주제는 TCP입니다. "3-way handshake가 뭔지 압니다" 수준에서 멈추지 않고,
스펙 → 커널 구현 → OS/하드웨어 경계 → 직접 만들어보기까지 순서대로 파고 내려갑니다.

## 확정 연재 (0~4단계)

매주 하나씩, 아래 순서로 발행합니다.

### 0단계 — 표면
1. 3-way handshake는 왜 이렇게 생겼나

### 1단계 — 스펙 (RFC)
2. RFC 793 vs RFC 9293, 뭐가 왜 바뀌었나
3. TCP 헤더 32비트 완전분해
4. 시퀀스 넘버 wraparound과 PAWS (RFC 1323)

### 2단계 — 커널 구현 (Linux 소스)
5. Linux TCP 상태머신, 소스로 따라가기
6. 소켓 버퍼·재전송 큐·RTO 계산의 실제 구현
7. 혼잡제어: Reno vs Cubic vs BBR 커널 코드 비교

### 3단계 — OS/하드웨어 경계
8. strace로 소켓 syscall 흐름 추적하기
9. NIC 오프로딩(TSO/GRO)과 epoll 내부 구현

### 4단계 — 직접 만들어서 확인 (뇌절의 정점)
10. Wireshark로 핸드셰이크 패킷 뜯어서 RFC와 대조하기
11. Raw socket으로 TCP 패킷 직접 만들기
12. 미니 TCP 상태머신 직접 구현하고 실제 커널과 비교
13. 혼잡제어 알고리즘을 시뮬레이션 코드로 구현해 처리량 비교

## 번외 뇌절 (내킬 때만)

여기부터는 매주 연재 약속은 없고, 필받을 때 하나씩 추가합니다.

- **물리 계층**: 케이블 위의 전압 파형, Manchester encoding, PHY 칩 동작
- **수학적 증명**: AIMD 공정성 증명(Chiu & Jain, 1989), RTO 공식(Jacobson/Karels, 1988) 유도
- **역사 고고학**: Cerf & Kahn 1974년 원논문, OSI vs TCP/IP 표준 전쟁, BSD Sockets 역사
- **형식 검증**: TCP 상태머신을 TLA+/Alloy로 명세하고 모델체커로 검증
- **대안 우주와 비교**: QUIC은 TCP의 어떤 문제를 다시 설계했나
- **하드웨어에 굽기**: FPGA로 미니 TCP/IP 오프로드 엔진 구현

이 로드맵은 진행하면서 계속 업데이트합니다.
