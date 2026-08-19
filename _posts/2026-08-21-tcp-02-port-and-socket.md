---
layout: post
title: "포트와 소켓 - 연결을 식별하는 방법"
date: 2026-08-21 09:00:00 +0900
categories: [CS]
tags: [뇌절, TCP, 기본]
series: TCP
tier: 기본
---

[지난 글]({% link _posts/2026-08-14-tcp-01-what-is-tcp.md %})에서 TCP는 IP 위에서 동작하며, 연결이라는 상태를 맺고 유지한다고 했습니다. 그런데 IP 주소 하나로 컴퓨터 한 대는 찾아갈 수 있어도, 그 컴퓨터 안에서 **어떤 프로그램**에게 데이터를 전달해야 하는지는 아직 정해지지 않았습니다. 이번 글은 그 문제를 다룹니다.

## IP 주소만으로는 왜 부족한가

서버 한 대(IP: `203.0.113.10`)에서 웹 서버, SSH, 메일 서버가 동시에 돌아가고 있다고 해봅시다. 인터넷에서 이 서버로 패킷이 하나 도착했습니다. IP 주소만 보면 "이 서버로 가는 패킷"이라는 것까지만 알 수 있고, 셋 중 누구에게 전달해야 하는지는 알 수 없습니다.

<figure class="post-figure">
<svg viewBox="0 0 640 250" role="img" aria-label="같은 IP 주소를 가진 호스트 안에 Web 서버(:80), SSH(:22), 메일 서버(:25)가 함께 떠 있다. 도착하는 패킷은 목적지 포트 번호로 어느 프로그램에게 갈지 결정된다." xmlns="http://www.w3.org/2000/svg">
  <defs>
    <marker id="tcp02-arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 z" fill="currentColor"/>
    </marker>
  </defs>

  <text x="40" y="20" font-size="12" fill="currentColor" fill-opacity="0.6" font-family="sans-serif">인터넷에서 도착하는 패킷</text>

  <rect x="380" y="15" width="230" height="220" fill="none" stroke="currentColor" stroke-width="1.5" rx="4"/>
  <text x="495" y="38" font-size="13" font-weight="600" fill="currentColor" font-family="sans-serif" text-anchor="middle">호스트 — IP 203.0.113.10</text>

  <rect x="400" y="54" width="190" height="52" fill="none" stroke="currentColor" stroke-width="1.5" rx="4"/>
  <text x="495" y="86" font-size="13" font-weight="600" fill="currentColor" font-family="sans-serif" text-anchor="middle">Web 서버 <tspan fill="var(--tcp-accent)">(:80)</tspan></text>

  <rect x="400" y="118" width="190" height="52" fill="none" stroke="currentColor" stroke-width="1.5" rx="4"/>
  <text x="495" y="150" font-size="13" font-weight="600" fill="currentColor" font-family="sans-serif" text-anchor="middle">SSH <tspan fill="var(--tcp-accent)">(:22)</tspan></text>

  <rect x="400" y="182" width="190" height="52" fill="none" stroke="currentColor" stroke-width="1.5" rx="4"/>
  <text x="495" y="214" font-size="13" font-weight="600" fill="currentColor" font-family="sans-serif" text-anchor="middle">메일 서버 <tspan fill="var(--tcp-accent)">(:25)</tspan></text>

  <line x1="40" y1="80" x2="398" y2="80" stroke="currentColor" stroke-width="1.5" marker-end="url(#tcp02-arrow)"/>
  <text x="215" y="72" font-size="11" fill="currentColor" fill-opacity="0.7" font-family="sans-serif" text-anchor="middle">목적지 203.0.113.10<tspan fill="var(--tcp-accent)">:80</tspan></text>

  <line x1="40" y1="144" x2="398" y2="144" stroke="currentColor" stroke-width="1.5" marker-end="url(#tcp02-arrow)"/>
  <text x="215" y="136" font-size="11" fill="currentColor" fill-opacity="0.7" font-family="sans-serif" text-anchor="middle">목적지 203.0.113.10<tspan fill="var(--tcp-accent)">:22</tspan></text>

  <line x1="40" y1="208" x2="398" y2="208" stroke="currentColor" stroke-width="1.5" marker-end="url(#tcp02-arrow)"/>
  <text x="215" y="200" font-size="11" fill="currentColor" fill-opacity="0.7" font-family="sans-serif" text-anchor="middle">목적지 203.0.113.10<tspan fill="var(--tcp-accent)">:25</tspan></text>
</svg>
<figcaption>IP는 호스트까지만 데려다주고, 그 안에서 어느 프로그램에게 전달할지는 포트 번호가 정한다.</figcaption>
</figure>

**포트(port)**는 이 문제를 푸는 16비트 숫자(0~65535)입니다. 패킷은 목적지 IP뿐 아니라 목적지 포트도 함께 가지고 다니고, 호스트는 그 포트 번호를 보고 어느 프로그램에게 넘길지 결정합니다.

## 포트 번호의 범위

포트 번호는 관행적으로 세 구간으로 나뉩니다.

| 구간 | 범위 | 용도 |
|---|---|---|
| Well-known | 0 ~ 1023 | HTTP(80), HTTPS(443), SSH(22)처럼 널리 합의된 서비스용. 대부분 OS에서 관리자 권한이 있어야 열 수 있음 |
| Registered | 1024 ~ 49151 | 특정 애플리케이션이 IANA에 등록해서 쓰는 포트 |
| Dynamic/Private | 49152 ~ 65535 | 클라이언트가 연결을 맺을 때 임시로 쓰는 포트. "임시 포트(ephemeral port)"라고도 부름 |

여기서 중요한 포인트: 웹 브라우저가 서버의 `:80`에 접속할 때, 브라우저 쪽도 포트가 필요합니다. 이때 브라우저는 보통 dynamic 구간에서 아무 포트나 하나 골라 씁니다. 그래서 서버 쪽은 항상 `:80`처럼 고정돼 있지만, 클라이언트 쪽 포트는 접속할 때마다 다릅니다. 이게 다음 이야기로 이어집니다.

## 포트만으로도 사실 부족하다 — 소켓과 5-tuple

여기서 자연스러운 의문이 생깁니다. 서버의 `:80`번 포트로 클라이언트 수천 명이 동시에 접속하면, 그 수천 개의 연결은 어떻게 서로 구분될까요? 다들 같은 포트로 들어오는데 말입니다.

답은 "포트 하나만으로 연결을 식별하지 않는다"입니다. TCP는 다음 다섯 가지 값의 조합, **5-tuple**로 각 연결을 구분합니다.

```
(프로토콜, 로컬 IP, 로컬 포트, 원격 IP, 원격 포트)
```

서버 입장에서 로컬 IP·로컬 포트(`203.0.113.10:80`)는 클라이언트가 몇 명이 붙든 항상 같습니다. 하지만 원격 IP·원격 포트(클라이언트 쪽)는 클라이언트마다 다릅니다. 그래서 5개 값을 통째로 보면 클라이언트 수만큼 서로 다른 조합이 나오고, 서버는 이 조합으로 각 연결을 구분합니다.

<figure class="post-figure">
<svg viewBox="0 0 640 240" role="img" aria-label="세 클라이언트가 모두 같은 서버의 같은 포트(203.0.113.10:80)로 접속하지만, 클라이언트 쪽 IP와 포트가 서로 달라서 TCP는 이 셋을 다른 연결로 구분한다." xmlns="http://www.w3.org/2000/svg">
  <defs>
    <marker id="tcp02b-arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 z" fill="currentColor"/>
    </marker>
  </defs>

  <rect x="20" y="20" width="190" height="48" fill="none" stroke="currentColor" stroke-width="1.5" rx="4"/>
  <text x="115" y="40" font-size="13" font-weight="600" fill="currentColor" font-family="sans-serif" text-anchor="middle">Client A</text>
  <text x="115" y="58" font-size="11" fill="currentColor" fill-opacity="0.7" font-family="sans-serif" text-anchor="middle">203.0.113.50:51000</text>

  <rect x="20" y="96" width="190" height="48" fill="none" stroke="currentColor" stroke-width="1.5" rx="4"/>
  <text x="115" y="116" font-size="13" font-weight="600" fill="currentColor" font-family="sans-serif" text-anchor="middle">Client B</text>
  <text x="115" y="134" font-size="11" fill="currentColor" fill-opacity="0.7" font-family="sans-serif" text-anchor="middle">198.51.100.7:52000</text>

  <rect x="20" y="172" width="190" height="48" fill="none" stroke="currentColor" stroke-width="1.5" rx="4"/>
  <text x="115" y="192" font-size="13" font-weight="600" fill="currentColor" font-family="sans-serif" text-anchor="middle">Client C</text>
  <text x="115" y="210" font-size="11" fill="currentColor" fill-opacity="0.7" font-family="sans-serif" text-anchor="middle">192.0.2.9:53000</text>

  <rect x="430" y="96" width="190" height="48" fill="none" stroke="var(--tcp-accent)" stroke-width="2.5" rx="4"/>
  <text x="525" y="116" font-size="13" font-weight="700" fill="var(--tcp-accent)" font-family="sans-serif" text-anchor="middle">Server</text>
  <text x="525" y="134" font-size="11" fill="var(--tcp-accent)" fill-opacity="0.9" font-family="sans-serif" text-anchor="middle">203.0.113.10:80</text>

  <line x1="212" y1="44" x2="428" y2="108" stroke="currentColor" stroke-width="1.5" marker-end="url(#tcp02b-arrow)"/>
  <line x1="212" y1="120" x2="428" y2="120" stroke="currentColor" stroke-width="1.5" marker-end="url(#tcp02b-arrow)"/>
  <line x1="212" y1="196" x2="428" y2="132" stroke="currentColor" stroke-width="1.5" marker-end="url(#tcp02b-arrow)"/>
</svg>
<figcaption>서버 쪽 IP:포트는 셋 다 같지만(203.0.113.10:80), 클라이언트 쪽 IP:포트가 다르기 때문에 TCP는 이 셋을 완전히 다른 연결로 구분한다.</figcaption>
</figure>

정리하면 이렇습니다.

- **포트**는 "같은 호스트 안 어느 프로그램인가"를 구분한다.
- **소켓**은 (프로토콜, IP, 포트) 조합으로, 한쪽 endpoint를 가리킨다.
- **5-tuple**(연결)은 로컬 소켓과 원격 소켓을 합친 것으로, TCP 연결 하나를 유일하게 식별한다.

"포트가 겹치면 안 되는 거 아닌가?"라는 흔한 오해는 여기서 풀립니다. 겹치면 안 되는 건 포트 번호가 아니라 5-tuple 전체입니다. 서버 쪽 포트(`:80`)는 모든 연결에서 똑같아도 되고, 실제로 항상 똑같습니다.

## 확인해보기

리눅스/macOS에서 `ss -tn`(또는 `netstat -tn`)을 실행하면 실제 5-tuple을 눈으로 볼 수 있습니다.

```
$ ss -tn
State   Local Address:Port     Peer Address:Port
ESTAB   192.168.0.5:22         203.0.113.50:51000
ESTAB   192.168.0.5:22         198.51.100.7:52000
```

`Local Address:Port`는 두 줄 다 같은데(SSH 데몬이 `:22`에서 듣고 있으니까), `Peer Address:Port`는 서로 다릅니다. 두 연결은 로컬 포트가 같아도 완전히 독립된, 서로 다른 TCP 연결입니다.

<div class="tangent" markdown="1">
## 곁다리: 집 공유기 뒤에서는 포트가 한 번 더 바뀐다

지금까지는 클라이언트가 쓰는 IP가 인터넷에 그대로 노출된다고 가정했습니다. 그런데 집 공유기(NAT) 뒤에 있는 기기들은 보통 사설 IP(`192.168.x.x` 같은)를 쓰고, 공인 IP는 공유기 하나만 가지고 있습니다. 같은 집 안의 노트북과 스마트폰이 동시에 같은 웹사이트에 접속하면, 인터넷에서 볼 때는 둘 다 "공유기의 공인 IP"에서 나온 것처럼 보입니다.

이걸 구분하는 방법도 결국 포트입니다. 공유기는 내부 기기가 쓰는 사설 IP:포트를 자기 자신의 공인 IP:포트로 바꿔치기해서 내보내고, 응답이 돌아오면 그 매핑표를 보고 원래 요청한 내부 기기에게 돌려줍니다. 결국 이 글의 핵심 아이디어("포트로 여러 개를 구분한다")가 여기서도 그대로 재사용되는 셈입니다 — 구분 대상만 "서버 안 여러 프로그램"에서 "공유기 뒤 여러 기기"로 바뀌었을 뿐입니다.
</div>

## 다음 글

이제 연결을 "식별하는 법"은 알았습니다. 다음 글에서는 이 5-tuple로 식별되는 연결이 실제로 어떻게 **맺어지는지** — 3-way handshake를 다룹니다.
