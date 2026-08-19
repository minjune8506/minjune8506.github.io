---
layout: post
title: "TCP란 무엇인가"
date: 2026-08-14 09:00:00 +0900
categories: [CS]
tags: [뇌절, TCP, 기본]
series: TCP
tier: 기본
---

**뇌절**의 첫 시리즈 주제, TCP입니다. [로드맵]({% link _posts/2026-08-13-tcp-series-roadmap.md %})에서 예고한 대로, 이번 글은 기본 단계의 첫 번째 글이라 아주 기초적인 질문부터 시작합니다: **TCP가 정확히 뭘 하는 프로토콜인가?**

## TCP는 어디에 있는가

인터넷에서 데이터를 주고받을 때, 여러 계층이 각자 역할을 나눠 맡습니다. TCP/IP 4계층 모델로 보면 이렇습니다.

<figure class="post-figure">
<svg viewBox="0 0 640 370" role="img" aria-label="TCP/IP 4계층 구조. Application 계층 아래 Transport 계층이 있고, Transport 계층 안에 TCP와 UDP가 나란히 있다. 그 아래 Internet 계층(IP), 가장 아래 Link 계층이 있다." xmlns="http://www.w3.org/2000/svg">
  <rect x="40" y="20" width="560" height="64" fill="none" stroke="currentColor" stroke-width="1.5" rx="4"/>
  <text x="60" y="58" font-size="14" font-weight="600" fill="currentColor" font-family="sans-serif">Application 계층</text>
  <text x="580" y="58" font-size="12" fill="currentColor" fill-opacity="0.6" font-family="sans-serif" text-anchor="end">HTTP · DNS · SSH ...</text>

  <rect x="40" y="98" width="560" height="100" fill="none" stroke="currentColor" stroke-width="1.5" rx="4"/>
  <text x="60" y="118" font-size="14" font-weight="600" fill="currentColor" font-family="sans-serif">Transport 계층</text>

  <rect x="60" y="130" width="250" height="54" fill="none" stroke="var(--tcp-accent)" stroke-width="2.5" rx="4"/>
  <text x="185" y="153" font-size="16" font-weight="700" fill="var(--tcp-accent)" font-family="sans-serif" text-anchor="middle">TCP</text>
  <text x="185" y="171" font-size="10" fill="var(--tcp-accent)" fill-opacity="0.85" font-family="sans-serif" text-anchor="middle">연결지향 · 신뢰성 — 이 시리즈의 주제</text>

  <rect x="330" y="130" width="250" height="54" fill="none" stroke="currentColor" stroke-width="1.5" rx="4"/>
  <text x="455" y="153" font-size="16" font-weight="700" fill="currentColor" font-family="sans-serif" text-anchor="middle">UDP</text>
  <text x="455" y="171" font-size="10" fill="currentColor" fill-opacity="0.6" font-family="sans-serif" text-anchor="middle">비연결 · 빠름</text>

  <rect x="40" y="212" width="560" height="64" fill="none" stroke="currentColor" stroke-width="1.5" rx="4"/>
  <text x="60" y="250" font-size="14" font-weight="600" fill="currentColor" font-family="sans-serif">Internet 계층</text>
  <text x="580" y="250" font-size="12" fill="currentColor" fill-opacity="0.6" font-family="sans-serif" text-anchor="end">IP</text>

  <rect x="40" y="290" width="560" height="64" fill="none" stroke="currentColor" stroke-width="1.5" rx="4"/>
  <text x="60" y="328" font-size="14" font-weight="600" fill="currentColor" font-family="sans-serif">Link 계층</text>
  <text x="580" y="328" font-size="12" fill="currentColor" fill-opacity="0.6" font-family="sans-serif" text-anchor="end">Ethernet · Wi-Fi ...</text>
</svg>
<figcaption>TCP는 Transport 계층에서 UDP와 나란히 동작하며, Internet 계층(IP) 위에 얹혀서 돈다.</figcaption>
</figure>

여기서 두 가지만 기억하면 됩니다.

1. **TCP는 IP 위에서 동작한다.** "TCP/IP"라는 이름 자체가 이 관계를 말해줍니다. IP는 "어디로 보낼지(주소)"만 책임지고, 그 패킷이 가는 중에 사라지든 순서가 뒤바뀌든 신경 쓰지 않습니다. 그 위에서 "확실하게, 순서대로" 보내는 역할을 TCP가 맡습니다.
2. **Transport 계층에는 TCP만 있는 게 아니다.** 같은 자리에 UDP도 있습니다. 둘은 같은 문제(애플리케이션 간에 데이터를 주고받는 것)를 완전히 다른 방식으로 풉니다. 이 차이가 이 글의 핵심입니다.

## "연결지향"과 "신뢰성"이 구체적으로 뭔가

TCP를 설명할 때 항상 따라붙는 두 단어가 "연결지향(connection-oriented)"과 "신뢰성 있는(reliable)"입니다. 각각 구체적으로 무슨 뜻인지 뜯어보겠습니다.

- **연결지향**: 데이터를 보내기 전에 먼저 양쪽이 "연결"이라는 상태를 만듭니다. 이 상태가 있어야만 데이터를 주고받을 수 있고, 다 끝나면 명시적으로 연결을 끊습니다. (연결을 어떻게 맺는지는 3편에서 다룹니다.)
- **신뢰성**: 보낸 데이터가 상대방에게 순서대로, 빠짐없이 도착했다는 걸 보장합니다. 중간에 유실되면 다시 보내고, 순서가 뒤바뀌어 도착해도 원래 순서로 재조립해줍니다. (구체적인 방법은 4~5편에서 다룹니다.)

UDP는 이 둘 다 하지 않습니다. 연결이라는 개념 자체가 없고, 보낸 데이터가 도착했는지도 UDP 스스로는 신경 쓰지 않습니다. 그림으로 비교하면 이렇습니다.

<figure class="post-figure">
<svg viewBox="0 0 640 130" role="img" aria-label="TCP는 Client와 Server가 연결을 맺고 유지하며 데이터를 주고받는다. UDP는 연결 없이 각 패킷을 독립적으로 전송한다." xmlns="http://www.w3.org/2000/svg">
  <defs>
    <marker id="tcp01-arrow-accent" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 z" fill="var(--tcp-accent)"/>
    </marker>
    <marker id="tcp01-arrow-plain" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 z" fill="currentColor"/>
    </marker>
  </defs>

  <text x="155" y="18" font-size="13" font-weight="600" fill="var(--tcp-accent)" font-family="sans-serif" text-anchor="middle">TCP — 연결지향</text>
  <rect x="30" y="46" width="90" height="40" fill="none" stroke="currentColor" stroke-width="1.5" rx="4"/>
  <text x="75" y="70" font-size="13" fill="currentColor" font-family="sans-serif" text-anchor="middle">Client</text>
  <rect x="190" y="46" width="90" height="40" fill="none" stroke="currentColor" stroke-width="1.5" rx="4"/>
  <text x="235" y="70" font-size="13" fill="currentColor" font-family="sans-serif" text-anchor="middle">Server</text>
  <line x1="122" y1="66" x2="188" y2="66" stroke="var(--tcp-accent)" stroke-width="4" marker-start="url(#tcp01-arrow-accent)" marker-end="url(#tcp01-arrow-accent)"/>
  <text x="155" y="112" font-size="11" fill="currentColor" fill-opacity="0.75" font-family="sans-serif" text-anchor="middle">연결을 맺고 유지하며 데이터 교환</text>

  <text x="480" y="18" font-size="13" font-weight="600" fill="currentColor" fill-opacity="0.85" font-family="sans-serif" text-anchor="middle">UDP — 비연결</text>
  <rect x="350" y="46" width="90" height="40" fill="none" stroke="currentColor" stroke-width="1.5" rx="4"/>
  <text x="395" y="70" font-size="13" fill="currentColor" font-family="sans-serif" text-anchor="middle">Client</text>
  <rect x="510" y="46" width="90" height="40" fill="none" stroke="currentColor" stroke-width="1.5" rx="4"/>
  <text x="555" y="70" font-size="13" fill="currentColor" font-family="sans-serif" text-anchor="middle">Server</text>
  <line x1="442" y1="54" x2="508" y2="54" stroke="currentColor" stroke-width="1.5" marker-end="url(#tcp01-arrow-plain)"/>
  <line x1="442" y1="66" x2="508" y2="66" stroke="currentColor" stroke-width="1.5" marker-end="url(#tcp01-arrow-plain)"/>
  <line x1="442" y1="78" x2="508" y2="78" stroke="currentColor" stroke-width="1.5" marker-end="url(#tcp01-arrow-plain)"/>
  <text x="480" y="112" font-size="11" fill="currentColor" fill-opacity="0.75" font-family="sans-serif" text-anchor="middle">그때그때 독립적으로 전송 (연결 없음)</text>
</svg>
<figcaption>TCP는 연결을 유지한 채로 데이터를 주고받고, UDP는 연결이라는 상태 없이 패킷을 독립적으로 보낸다.</figcaption>
</figure>

UDP 쪽 화살표 3개가 서로 이어져 있지 않다는 점이 중요합니다. 각 패킷은 서로 아무 관계가 없습니다. 하나가 중간에 사라져도 UDP는 그걸 모르고, 순서가 뒤바뀌어 도착해도 신경 쓰지 않습니다. 그 대신 "연결을 맺고 유지하는" 오버헤드가 없어서 더 가볍고 빠릅니다.

## TCP vs UDP

| | TCP | UDP |
|---|---|---|
| 연결 | 연결지향 (핸드셰이크 필요) | 비연결 |
| 신뢰성 | 순서 보장, 유실 시 재전송 | 보장 없음 |
| 속도 | 상대적으로 느림 (연결 유지·재전송 비용) | 빠름 |
| 오버헤드 | 큼 (헤더 20바이트, 상태 관리) | 작음 (헤더 8바이트) |
| 대표 사용처 | 웹(HTTP), 파일 전송, 이메일 — "틀리면 안 되는" 데이터 | 실시간 스트리밍, 온라인 게임, DNS — "늦으면 의미 없는" 데이터 |

이 표에서 가장 중요한 건 "TCP가 항상 더 좋다"가 아니라는 겁니다. 실시간 음성 통화에서는 0.5초 전 패킷이 지금 도착해서 재생되는 것보다, 그냥 버리고 최신 패킷을 트는 게 낫습니다. TCP가 유실된 패킷을 재전송하려고 애쓰는 동안 통화는 이미 끊긴 것처럼 느껴집니다. 그래서 이런 경우엔 UDP를 씁니다. 반대로 파일 하나를 내려받는데 중간 바이트 몇 개가 빠지면 파일 자체가 깨지므로, 이런 경우엔 반드시 TCP를 씁니다.

<div class="tangent" markdown="1">
## 곁다리: TCP가 없다면?

간단한 사고실험을 해봅시다. TCP 없이 IP 위에 애플리케이션을 바로 올린다면, 개발자가 직접 만들어야 하는 목록이 이렇게 됩니다.

- 패킷이 유실됐는지 확인하고 다시 보내는 로직
- 순서가 뒤바뀌어 도착한 패킷을 원래 순서로 재조립하는 로직
- 수신자가 처리를 못 따라가면 속도를 늦추는 로직
- 네트워크가 혼잡하면 속도를 늦추는 로직
- 상대방이 아직 연결돼 있는지 확인하는 로직

TCP는 이 전부를 대신 해주는 하나의 "패키지"입니다. 사실 이 시리즈 남은 글들은 이 목록을 하나씩 뜯어보는 것과 다르지 않습니다 — 4편은 흐름제어, 5편은 재전송·재조립, 6편은 혼잡제어입니다. 이 목록을 기억해두면, 앞으로 나올 각 메커니즘이 "왜 존재하는지"가 뻔하게 느껴질 겁니다.
</div>

## 이 시리즈에서 쓸 용어

앞으로 자주 나올 단어 세 개를 미리 정리합니다.

- **세그먼트(segment)**: TCP가 주고받는 데이터 단위. (IP 계층에서는 이걸 "패킷", Link 계층에서는 "프레임"이라고 부릅니다. 같은 데이터를 계층마다 다르게 부르는 것뿐입니다.)
- **연결(connection)**: 3-way handshake로 맺어지고 4-way handshake로 끊어지는, 두 endpoint 사이의 상태. TCP의 모든 동작은 이 연결이 있다는 걸 전제로 합니다.
- **스트림(stream)**: TCP가 애플리케이션에 제공하는 추상화. 애플리케이션 입장에서 TCP는 "세그먼트를 여러 개 주고받는 것"이 아니라 "끊김 없는 바이트의 흐름"처럼 보입니다. 이 스트림 추상화 뒤에서 세그먼트 분할, 재조립, 재전송이 전부 숨겨져 있습니다.

## 다음 글

지금까지는 "TCP가 뭘 하는가"를 큰 그림으로만 봤습니다. 다음 글에서는 "같은 컴퓨터에서 여러 프로그램이 동시에 TCP를 쓰면 어떻게 구분하는가" — 포트와 소켓 이야기로 넘어갑니다.
