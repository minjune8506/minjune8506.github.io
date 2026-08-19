---
layout: post
title: "09. TCP 헤더 32비트 완전분해"
date: 2026-10-09 09:00:00 +0900
categories: [CS]
tags: [뇌절, TCP, 심화, RFC]
series: TCP
tier: 심화
mermaid: false
---

[지난 글]({% link _posts/2026-10-02-tcp-08-rfc793-vs-rfc9293.md %})에서 스펙 본문이 어떻게 바뀌었는지 봤습니다. 이번엔 시선을 좁혀서, TCP 헤더 20바이트를 비트 단위로 완전히 분해합니다. RFC 9293 Section 3.1의 다이어그램을 그대로 따라갑니다.

## 헤더 전체 구조

<figure class="post-figure">
<svg viewBox="0 0 660 340" role="img" aria-label="TCP 헤더 20바이트를 32비트씩 6줄로 나눈 구조. Source/Destination Port, Sequence Number, Acknowledgment Number, Data Offset·Reserved·제어비트·Window, Checksum·Urgent Pointer, 그리고 가변 길이 Options 순서로 배치된다." xmlns="http://www.w3.org/2000/svg">
  <text x="20" y="16" font-size="9" fill="currentColor" fill-opacity="0.5" font-family="sans-serif">0</text>
  <text x="320" y="16" font-size="9" fill="currentColor" fill-opacity="0.5" font-family="sans-serif" text-anchor="middle">16</text>
  <text x="620" y="16" font-size="9" fill="currentColor" fill-opacity="0.5" font-family="sans-serif" text-anchor="end">31</text>

  <rect x="20" y="24" width="300" height="42" fill="none" stroke="currentColor" stroke-width="1.5"/>
  <text x="170" y="49" font-size="12" fill="currentColor" font-family="sans-serif" text-anchor="middle">Source Port (16)</text>
  <rect x="320" y="24" width="300" height="42" fill="none" stroke="currentColor" stroke-width="1.5"/>
  <text x="470" y="49" font-size="12" fill="currentColor" font-family="sans-serif" text-anchor="middle">Destination Port (16)</text>

  <rect x="20" y="72" width="600" height="42" fill="none" stroke="currentColor" stroke-width="1.5"/>
  <text x="320" y="97" font-size="12" fill="currentColor" font-family="sans-serif" text-anchor="middle">Sequence Number (32)</text>

  <rect x="20" y="120" width="600" height="42" fill="none" stroke="currentColor" stroke-width="1.5"/>
  <text x="320" y="145" font-size="12" fill="currentColor" font-family="sans-serif" text-anchor="middle">Acknowledgment Number (32)</text>

  <rect x="20" y="168" width="75" height="42" fill="none" stroke="currentColor" stroke-width="1.5"/>
  <text x="57" y="184" font-size="9" fill="currentColor" font-family="sans-serif" text-anchor="middle">Data</text>
  <text x="57" y="196" font-size="9" fill="currentColor" font-family="sans-serif" text-anchor="middle">Offset (4)</text>

  <rect x="95" y="168" width="75" height="42" fill="none" stroke="currentColor" stroke-width="1.5"/>
  <text x="132" y="184" font-size="9" fill="currentColor" font-family="sans-serif" text-anchor="middle">Reserved</text>
  <text x="132" y="196" font-size="9" fill="currentColor" font-family="sans-serif" text-anchor="middle">(4)</text>

  <rect x="170" y="168" width="150" height="42" fill="none" stroke="var(--tcp-accent)" stroke-width="2"/>
  <text x="245" y="184" font-size="9" fill="var(--tcp-accent)" font-family="sans-serif" text-anchor="middle">CWR ECE URG ACK</text>
  <text x="245" y="196" font-size="9" fill="var(--tcp-accent)" font-family="sans-serif" text-anchor="middle">PSH RST SYN FIN</text>

  <rect x="320" y="168" width="300" height="42" fill="none" stroke="currentColor" stroke-width="1.5"/>
  <text x="470" y="193" font-size="12" fill="currentColor" font-family="sans-serif" text-anchor="middle">Window (16)</text>

  <rect x="20" y="216" width="300" height="42" fill="none" stroke="currentColor" stroke-width="1.5"/>
  <text x="170" y="241" font-size="12" fill="currentColor" font-family="sans-serif" text-anchor="middle">Checksum (16)</text>
  <rect x="320" y="216" width="300" height="42" fill="none" stroke="currentColor" stroke-width="1.5"/>
  <text x="470" y="241" font-size="12" fill="currentColor" font-family="sans-serif" text-anchor="middle">Urgent Pointer (16)</text>

  <rect x="20" y="264" width="600" height="42" fill="none" stroke="currentColor" stroke-width="1.5" stroke-dasharray="5 3"/>
  <text x="320" y="282" font-size="12" fill="currentColor" font-family="sans-serif" text-anchor="middle">Options (0~40바이트, 4바이트 단위, 없을 수도 있음)</text>
  <text x="320" y="298" font-size="10" fill="currentColor" fill-opacity="0.6" font-family="sans-serif" text-anchor="middle">MSS · Window Scale · SACK Permitted · Timestamps ...</text>
</svg>
<figcaption>TCP 헤더는 최소 20바이트(옵션 없을 때), 32비트씩 5줄 + 가변 길이 Options로 구성된다.</figcaption>
</figure>

기본 20바이트(옵션 없이)에 왜 하필 이 필드들이 이 순서로 있는지, 하나씩 봅니다.

- **Source/Destination Port (16+16비트)**: 2편에서 본 그 포트입니다. 16비트라서 0~65535 범위가 나옵니다.
- **Sequence Number / Acknowledgment Number (32+32비트)**: 3~5편에서 계속 나온 그 숫자들입니다. 32비트라서 표현 범위가 약 43억이고, 이게 한 바퀴 도는 문제를 다음 글(10편)에서 다룹니다.
- **Data Offset (4비트)**: 헤더 전체 길이를 32비트(4바이트) 단위로 표현합니다. 옵션이 없으면 20바이트 = 4바이트×5 이므로 값은 5. 옵션이 붙으면 이 값이 커집니다. 4비트라서 최댓값은 15 — 즉 TCP 헤더는 아무리 옵션이 많아도 15×4=60바이트를 못 넘습니다.
- **Reserved (4비트)**: 미래를 위해 비워둔 자리. 원래 RFC 793에서는 6비트였는데, 그중 2비트가 지금 보이는 CWR·ECE로 옮겨갔습니다. 바로 다음 섹션에서 자세히 봅니다.
- **Window (16비트)**: 4편의 그 rwnd입니다. 16비트라서 최대로 표현 가능한 값은 65535바이트인데, 실제로는 이보다 큰 윈도우가 필요할 때가 많아서 Window Scale 옵션으로 이 한계를 우회합니다(아래 옵션 섹션에서 다룹니다).
- **Checksum (16비트)**: 헤더와 데이터가 전송 중 손상되지 않았는지 확인하는 값. TCP 헤더뿐 아니라 IP 주소까지 포함한 "pseudo header"를 더해서 계산합니다 — 그래서 IP 계층 정보 없이는 TCP 체크섬을 검증할 수 없습니다.
- **Urgent Pointer (16비트)**: 8편에서 본 그 필드. URG 비트가 켜져 있을 때만 의미가 있습니다.

## 제어 비트 8개, 확대해서 보기

위 다이어그램에서 강조된 8비트를 하나씩 뜯어보면 이렇습니다.

<figure class="post-figure">
<svg viewBox="0 0 660 140" role="img" aria-label="8개 제어 비트: CWR, ECE, URG, ACK, PSH, RST, SYN, FIN. CWR와 ECE는 원래 Reserved였다가 ECN용으로 재할당된 비트다." xmlns="http://www.w3.org/2000/svg">
  <rect x="18" y="15" width="72" height="50" fill="none" stroke="var(--tcp-accent)" stroke-width="2"/>
  <text x="54" y="45" font-size="13" font-weight="700" fill="var(--tcp-accent)" font-family="sans-serif" text-anchor="middle">CWR</text>
  <text x="54" y="80" font-size="9" fill="currentColor" fill-opacity="0.75" font-family="sans-serif" text-anchor="middle">혼잡 줄임</text>
  <text x="54" y="92" font-size="9" fill="currentColor" fill-opacity="0.75" font-family="sans-serif" text-anchor="middle">알림</text>

  <rect x="94" y="15" width="72" height="50" fill="none" stroke="var(--tcp-accent)" stroke-width="2"/>
  <text x="130" y="45" font-size="13" font-weight="700" fill="var(--tcp-accent)" font-family="sans-serif" text-anchor="middle">ECE</text>
  <text x="130" y="80" font-size="9" fill="currentColor" fill-opacity="0.75" font-family="sans-serif" text-anchor="middle">ECN 신호</text>
  <text x="130" y="92" font-size="9" fill="currentColor" fill-opacity="0.75" font-family="sans-serif" text-anchor="middle">받음</text>

  <rect x="170" y="15" width="72" height="50" fill="none" stroke="currentColor" stroke-width="1.5"/>
  <text x="206" y="45" font-size="13" font-weight="700" fill="currentColor" font-family="sans-serif" text-anchor="middle">URG</text>
  <text x="206" y="80" font-size="9" fill="currentColor" fill-opacity="0.75" font-family="sans-serif" text-anchor="middle">긴급 데이터</text>
  <text x="206" y="92" font-size="9" fill="currentColor" fill-opacity="0.75" font-family="sans-serif" text-anchor="middle">있음</text>

  <rect x="246" y="15" width="72" height="50" fill="none" stroke="currentColor" stroke-width="1.5"/>
  <text x="282" y="45" font-size="13" font-weight="700" fill="currentColor" font-family="sans-serif" text-anchor="middle">ACK</text>
  <text x="282" y="80" font-size="9" fill="currentColor" fill-opacity="0.75" font-family="sans-serif" text-anchor="middle">확인 번호</text>
  <text x="282" y="92" font-size="9" fill="currentColor" fill-opacity="0.75" font-family="sans-serif" text-anchor="middle">유효함</text>

  <rect x="322" y="15" width="72" height="50" fill="none" stroke="currentColor" stroke-width="1.5"/>
  <text x="358" y="45" font-size="13" font-weight="700" fill="currentColor" font-family="sans-serif" text-anchor="middle">PSH</text>
  <text x="358" y="80" font-size="9" fill="currentColor" fill-opacity="0.75" font-family="sans-serif" text-anchor="middle">즉시 위로</text>
  <text x="358" y="92" font-size="9" fill="currentColor" fill-opacity="0.75" font-family="sans-serif" text-anchor="middle">전달</text>

  <rect x="398" y="15" width="72" height="50" fill="none" stroke="currentColor" stroke-width="1.5"/>
  <text x="434" y="45" font-size="13" font-weight="700" fill="currentColor" font-family="sans-serif" text-anchor="middle">RST</text>
  <text x="434" y="80" font-size="9" fill="currentColor" fill-opacity="0.75" font-family="sans-serif" text-anchor="middle">강제</text>
  <text x="434" y="92" font-size="9" fill="currentColor" fill-opacity="0.75" font-family="sans-serif" text-anchor="middle">재설정</text>

  <rect x="474" y="15" width="72" height="50" fill="none" stroke="currentColor" stroke-width="1.5"/>
  <text x="510" y="45" font-size="13" font-weight="700" fill="currentColor" font-family="sans-serif" text-anchor="middle">SYN</text>
  <text x="510" y="80" font-size="9" fill="currentColor" fill-opacity="0.75" font-family="sans-serif" text-anchor="middle">연결</text>
  <text x="510" y="92" font-size="9" fill="currentColor" fill-opacity="0.75" font-family="sans-serif" text-anchor="middle">시작</text>

  <rect x="550" y="15" width="72" height="50" fill="none" stroke="currentColor" stroke-width="1.5"/>
  <text x="586" y="45" font-size="13" font-weight="700" fill="currentColor" font-family="sans-serif" text-anchor="middle">FIN</text>
  <text x="586" y="80" font-size="9" fill="currentColor" fill-opacity="0.75" font-family="sans-serif" text-anchor="middle">연결</text>
  <text x="586" y="92" font-size="9" fill="currentColor" fill-opacity="0.75" font-family="sans-serif" text-anchor="middle">종료</text>

  <text x="330" y="122" font-size="10" fill="currentColor" fill-opacity="0.6" font-family="sans-serif" text-anchor="middle">헤더에 실제 등장하는 순서 그대로 (RFC 9293 Section 3.1)</text>
</svg>
<figcaption>CWR·ECE(강조)는 원래 Reserved 비트였다가 RFC 3168(ECN)에서 재할당됐다. 나머지 6개는 RFC 793 원본부터 있던 비트다.</figcaption>
</figure>

절반은 이미 낯이 익습니다. **SYN**·**FIN**은 3편·7편에서 연결을 맺고 끊을 때 봤고, **ACK**는 5편에서 재전송을, **RST**는 3편·8편에서 SYN flood와 스푸핑 방어를 다룰 때 나왔습니다. **URG**는 8편에서 urgent pointer와 함께, **PSH**는 이번이 처음인데 — 수신자의 TCP 버퍼가 데이터를 모아뒀다가 애플리케이션에 넘기는 대신, 이 비트가 켜져 있으면 지금까지 쌓인 걸 즉시 위로(애플리케이션 계층으로) 넘기라는 신호입니다.

새로 나온 건 **CWR**과 **ECE**입니다. 6편 곁다리에서 "라우터가 IP 헤더에 혼잡 조짐을 표시하고, 수신자는 그걸 ACK에 실어 보낸다"고 했는데, 그 "ACK에 싣는" 부분이 바로 이 두 비트입니다. 수신자가 라우터의 ECN 표시를 발견하면 **ECE**를 켜서 송신자에게 "혼잡 신호 받았어"라고 알리고, 송신자는 cwnd를 줄인 뒤 **CWR**을 켜서 "나 줄였어, 그만 알려줘도 돼"라고 응답합니다. 이 두 비트가 RFC 793 시절엔 그냥 "Reserved"로 비어 있었다는 걸 생각하면, "예약 비트는 미래를 위해 비워둔다"는 말이 정확히 이런 식으로 실현된 셈입니다.

## Options — 옵션 필드는 왜 있는가

Data Offset이 5보다 크면, Acknowledgment Number와 실제 데이터 사이에 Options 영역이 끼어듭니다. 옵션이 존재하는 이유는 간단합니다 — **고정 20바이트 헤더를 한 번 정해놓고 나니, 나중에 뭔가 새로 추가하고 싶을 때 헤더 구조 자체를 바꿀 수 없었기 때문**입니다. 대신 "옵션"이라는 확장 슬롯을 만들어서, 필요한 기능만 골라 붙이는 방식을 택했습니다. 지금까지 나온 옵션 중 이 시리즈와 관련된 것만 추리면:

- **MSS(Maximum Segment Size)**: handshake의 SYN에만 실려서, "나는 한 세그먼트에 최대 이만큼만 담을 수 있어"를 상대에게 알립니다.
- **Window Scale**: Window 필드가 16비트라 65535바이트가 한계라고 했는데, 이 옵션은 실제 윈도우 값에 곱할 배율(shift count)을 알려줘서 훨씬 큰 윈도우를 쓸 수 있게 해줍니다. 4편에서 다룬 rwnd가 사실은 이 옵션 덕분에 65535바이트보다 커질 수 있는 겁니다.
- **SACK Permitted / SACK**: 5편 곁다리에서 다룬 그 SACK입니다. handshake 때 "나 SACK 지원해"를 SACK Permitted 옵션으로 확인하고, 이후 실제 데이터 전송 중에는 SACK 옵션에 "이 구간도 받았어"를 실어 보냅니다.
- **Timestamps**: 세그먼트를 보낼 때 타임스탬프를 같이 실어서, RTT(왕복 시간)를 정밀하게 측정할 수 있게 해줍니다. RTO 계산(12편에서 다룰 예정)의 정확도가 여기 달려 있습니다.

## 실전: 헥사덤프 하나 읽어보기

지금까지 배운 걸 직접 써먹어봅니다. 다음은 옵션 없는(20바이트) SYN 세그먼트의 헤더를 16진수로 나타낸 것입니다.

```
C7 38 01 BB 00 00 03 E8 00 00 00 00 50 02 FA F0 12 34 00 00
```

바이트 위치를 따라가며 해석하면:

| 바이트 | 값 | 의미 |
|---|---|---|
| 0~1 | `C7 38` | Source Port = 0xC738 = 51000 |
| 2~3 | `01 BB` | Destination Port = 0x01BB = 443 |
| 4~7 | `00 00 03 E8` | Sequence Number = 1000 |
| 8~11 | `00 00 00 00` | Acknowledgment Number = 0 (아직 확인할 게 없음 — SYN이니까) |
| 12 | `50` | Data Offset=5(상위 4비트), Reserved=0(하위 4비트) → `0101 0000` |
| 13 | `02` | 제어 비트 = `0000 0010` → SYN만 켜짐 |
| 14~15 | `FA F0` | Window = 0xFAF0 = 64240 |
| 16~17 | `12 34` | Checksum (IP 주소까지 포함해서 계산하므로 여기서는 임의 값) |
| 18~19 | `00 00` | Urgent Pointer = 0 (URG 꺼져 있으니 의미 없음) |

목적지 포트가 443(HTTPS), 시퀀스 넘버가 1000, SYN 비트만 켜져 있는 걸 보면 — 3편에서 본 handshake의 첫 번째 메시지, 그것도 우리가 1편에서 처음 그린 그림 그대로라는 걸 바이트 레벨에서 확인할 수 있습니다.

## 다음 글

Sequence Number가 32비트라는 걸 이번 글에서 확인했습니다. 32비트면 언젠가 최댓값을 넘어서 다시 0으로 돌아갈 텐데, 그때 무슨 일이 일어날까요? 다음 글에서 wraparound과 PAWS를 다룹니다.
