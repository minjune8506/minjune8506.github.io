---
layout: post
title: "06. Congestion Control"
date: 2026-09-18 09:00:00 +0900
categories: [CS]
tags: [뇌절, TCP, 기본, 혼잡제어]
series: TCP
tier: 기본
---

[지난 글]({% link _posts/2026-09-11-tcp-05-reliability-ack-retransmission.md %})에서 "패킷이 유실되면 재전송한다"까지 다뤘습니다. 그런데 애초에 왜 유실이 일어날까요? 대부분은 네트워크 중간의 라우터가 처리 용량을 넘는 트래픽을 받아서 패킷을 그냥 버리기 때문입니다. 이번 글은 이 문제, **혼잡제어(congestion control)**를 다룹니다.

## 흐름제어와 혼잡제어, 뭐가 다른가

이름이 비슷해서 자주 헷갈리는데, 둘은 **보호하는 대상이 다릅니다.**

<figure class="post-figure">
<svg viewBox="0 0 640 220" role="img" aria-label="흐름제어는 수신자 한 명의 버퍼를 보호하고, 혼잡제어는 여러 연결이 함께 쓰는 공유 회선을 보호한다." xmlns="http://www.w3.org/2000/svg">
  <defs>
    <marker id="tcp06-arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 z" fill="currentColor"/>
    </marker>
  </defs>

  <text x="160" y="20" font-size="13" font-weight="600" fill="currentColor" font-family="sans-serif" text-anchor="middle">흐름제어 — 수신자 버퍼 보호</text>
  <rect x="40" y="94" width="90" height="36" fill="none" stroke="currentColor" stroke-width="1.5" rx="4"/>
  <text x="85" y="117" font-size="12" fill="currentColor" font-family="sans-serif" text-anchor="middle">Sender</text>
  <rect x="190" y="94" width="90" height="36" fill="none" stroke="var(--tcp-accent)" stroke-width="2.5" rx="4"/>
  <text x="235" y="117" font-size="12" fill="var(--tcp-accent)" font-family="sans-serif" text-anchor="middle">Receiver</text>
  <line x1="132" y1="112" x2="188" y2="112" stroke="currentColor" stroke-width="1.5" marker-end="url(#tcp06-arrow)"/>
  <text x="160" y="160" font-size="11" fill="currentColor" fill-opacity="0.7" font-family="sans-serif" text-anchor="middle">rwnd로 이 한 연결의</text>
  <text x="160" y="176" font-size="11" fill="currentColor" fill-opacity="0.7" font-family="sans-serif" text-anchor="middle">속도만 조절</text>

  <text x="480" y="20" font-size="13" font-weight="600" fill="currentColor" font-family="sans-serif" text-anchor="middle">혼잡제어 — 공유 회선 보호</text>
  <rect x="340" y="40" width="70" height="32" fill="none" stroke="currentColor" stroke-width="1.5" rx="4"/>
  <text x="375" y="61" font-size="10" fill="currentColor" font-family="sans-serif" text-anchor="middle">Sender A</text>
  <rect x="340" y="148" width="70" height="32" fill="none" stroke="currentColor" stroke-width="1.5" rx="4"/>
  <text x="375" y="169" font-size="10" fill="currentColor" font-family="sans-serif" text-anchor="middle">Sender B</text>
  <rect x="430" y="94" width="80" height="32" fill="none" stroke="var(--tcp-accent)" stroke-width="2.5" rx="4"/>
  <text x="470" y="115" font-size="10" fill="var(--tcp-accent)" font-family="sans-serif" text-anchor="middle">공유 회선</text>
  <rect x="540" y="40" width="70" height="32" fill="none" stroke="currentColor" stroke-width="1.5" rx="4"/>
  <text x="575" y="61" font-size="10" fill="currentColor" font-family="sans-serif" text-anchor="middle">Receiver A</text>
  <rect x="540" y="148" width="70" height="32" fill="none" stroke="currentColor" stroke-width="1.5" rx="4"/>
  <text x="575" y="169" font-size="10" fill="currentColor" font-family="sans-serif" text-anchor="middle">Receiver B</text>

  <line x1="410" y1="56" x2="430" y2="102" stroke="currentColor" stroke-width="1.5" marker-end="url(#tcp06-arrow)"/>
  <line x1="510" y1="102" x2="540" y2="56" stroke="currentColor" stroke-width="1.5" marker-end="url(#tcp06-arrow)"/>
  <line x1="410" y1="164" x2="430" y2="118" stroke="currentColor" stroke-width="1.5" marker-end="url(#tcp06-arrow)"/>
  <line x1="510" y1="118" x2="540" y2="164" stroke="currentColor" stroke-width="1.5" marker-end="url(#tcp06-arrow)"/>

  <text x="480" y="200" font-size="11" fill="currentColor" fill-opacity="0.7" font-family="sans-serif" text-anchor="middle">여러 연결이 함께 쓰는 대역폭을 조절</text>
</svg>
<figcaption>흐름제어는 연결 하나의 수신자 버퍼를 보호하고, 혼잡제어는 여러 연결이 함께 쓰는 네트워크 구간(라우터·회선)을 보호한다.</figcaption>
</figure>

흐름제어는 "상대방(수신자)이 감당할 수 있는가"만 신경 씁니다. 혼잡제어는 "이 연결과 무관한 다른 연결들이 같은 회선을 쓰고 있는데, 그 회선 자체가 감당할 수 있는가"를 신경 씁니다. 그래서 혼잡제어는 나와 상관없는 다른 트래픽의 상태까지 간접적으로 고려해야 하고, 상대방이 명시적으로 알려주는 rwnd와 달리 **네트워크가 알려주지 않는 값을 스스로 추정**해야 합니다.

## 혼잡 윈도우(cwnd) — 또 하나의 제한

흐름제어에 rwnd가 있었다면, 혼잡제어에는 **cwnd(congestion window)**가 있습니다. 송신자는 이제 두 가지 제한을 동시에 지켜야 합니다.

```
실제로 보낼 수 있는 양 = min(cwnd, rwnd)
```

rwnd는 상대방이 ACK에 실어서 알려주는 값이지만, cwnd는 네트워크가 알려주는 값이 없으므로 **송신자가 스스로 추정**해서 조절합니다. "네트워크가 감당할 수 있는 만큼"이라는 게 정확히 얼마인지는 아무도 알려주지 않으니, TCP는 조금씩 늘려보다가 문제(유실)가 생기면 줄이는 식으로 값을 찾아갑니다.

## AIMD — 느리게 늘리고 빠르게 줄인다

이 "조금씩 늘려보다가, 문제가 생기면 줄인다"는 전략을 **AIMD(Additive Increase, Multiplicative Decrease)**라고 부릅니다. cwnd가 시간에 따라 어떻게 움직이는지 그래프로 보면 이렇습니다.

<figure class="post-figure">
<svg viewBox="0 0 640 300" role="img" aria-label="cwnd는 slow start 구간에서 지수적으로 빠르게 증가하다가 ssthresh에 도달하면 congestion avoidance 구간에서 선형으로 천천히 증가하고, 패킷 유실이 감지되면 절반으로 뚝 떨어진 뒤 다시 선형 증가를 반복한다." xmlns="http://www.w3.org/2000/svg">
  <line x1="40" y1="30" x2="40" y2="260" stroke="currentColor" stroke-width="1.5"/>
  <line x1="40" y1="260" x2="600" y2="260" stroke="currentColor" stroke-width="1.5"/>
  <text x="20" y="145" font-size="12" fill="currentColor" font-family="sans-serif" text-anchor="middle" transform="rotate(-90 20 145)">cwnd</text>
  <text x="580" y="280" font-size="12" fill="currentColor" fill-opacity="0.7" font-family="sans-serif" text-anchor="middle">시간 (RTT 경과)</text>

  <line x1="40" y1="120" x2="200" y2="120" stroke="currentColor" stroke-width="1" stroke-dasharray="3 3" fill-opacity="0.6"/>
  <text x="30" y="124" font-size="10" fill="currentColor" fill-opacity="0.6" font-family="sans-serif" text-anchor="end">ssthresh</text>

  <polyline points="60,250 90,240 110,220 130,190 150,155 170,130 185,122 200,120" fill="none" stroke="currentColor" stroke-width="2"/>
  <text x="100" y="235" font-size="11" fill="currentColor" fill-opacity="0.75" font-family="sans-serif" text-anchor="middle">Slow Start</text>

  <line x1="200" y1="120" x2="380" y2="50" stroke="currentColor" stroke-width="2"/>
  <text x="290" y="80" font-size="11" fill="currentColor" fill-opacity="0.75" font-family="sans-serif" text-anchor="middle">Congestion Avoidance</text>

  <line x1="380" y1="50" x2="380" y2="150" stroke="var(--tcp-warn)" stroke-width="1.5" stroke-dasharray="4 3"/>
  <circle cx="380" cy="50" r="4" fill="var(--tcp-warn)"/>
  <text x="380" y="35" font-size="11" font-weight="600" fill="var(--tcp-warn)" font-family="sans-serif" text-anchor="middle">패킷 유실 감지</text>
  <text x="440" y="105" font-size="11" fill="var(--tcp-warn)" font-family="sans-serif" text-anchor="middle">cwnd 절반으로</text>

  <line x1="380" y1="150" x2="580" y2="95" stroke="currentColor" stroke-width="2"/>
</svg>
<figcaption>Slow start에서 지수적으로 빠르게 늘리다가 ssthresh 근처부터는 congestion avoidance로 전환해 선형으로 천천히 늘린다. 유실이 감지되면 cwnd가 뚝 떨어지고 같은 패턴이 반복된다 — 이 모양 때문에 흔히 "톱니(sawtooth)"라고 부른다.</figcaption>
</figure>

- **Slow start**: 연결 초반에는 네트워크 상태를 전혀 모르므로, 작게 시작해서 ACK를 받을 때마다 cwnd를 빠르게(지수적으로) 늘립니다. 이름은 "느리게 시작"이지만 증가 속도 자체는 가장 빠른 구간입니다 — 이전에 아무것도 안 보내던 것에 비하면 "조심스럽게 시작한다"는 뜻입니다.
- **ssthresh(slow start threshold)**: 이쯤부터는 슬슬 조심하자는 기준선입니다. cwnd가 여기 도달하면 증가 방식을 바꿉니다.
- **Congestion avoidance**: ssthresh를 넘으면 증가 속도를 늦춰서, RTT마다 조금씩(선형으로)만 늘립니다. 한계에 가까워졌으니 천천히 탐색하는 겁니다.
- **유실 발생**: 어딘가에서 패킷이 유실되면 "한계를 넘었다"는 신호로 해석하고, cwnd를 큰 폭으로(전형적으로 절반) 줄입니다. 그리고 다시 늘리기 시작합니다.

이 전체 사이클이 계속 반복되면서 톱니 모양 그래프가 그려집니다. "느리게 늘리고 빠르게 줄인다(AIMD)"는 이름 그대로, 늘릴 때는 조심스럽게 선형으로, 줄일 때는 과감하게 한 방에 줄입니다.

<div class="tangent" markdown="1">
## 유실될 때까지 기다려야만 할까 — ECN

지금까지 본 방식은 전부 "패킷을 잃고 나서야" 혼잡을 알아챕니다. 좀 아깝지 않나요? 라우터가 자기 큐가 차오르고 있다는 걸 미리 알면서도, 굳이 패킷을 버릴 때까지 기다렸다가 그제서야 송신자에게 신호를 주는 셈이니까요.

**ECN(Explicit Congestion Notification)**은 이걸 미리 알려주는 방법입니다. 라우터가 큐가 찰 것 같으면 패킷을 버리는 대신 IP 헤더의 ECN 비트에 "혼잡 조짐 있음" 표시만 해서 그대로 보냅니다. 수신자는 이 표시를 보고 ACK에 "혼잡 신호 받았다"는 걸 실어 보내고, 송신자는 패킷을 하나도 잃지 않고도 cwnd를 줄일 수 있습니다. 손실 기반 방식보다 훨씬 빠르고, 손해(재전송) 없이 혼잡을 감지하는 셈입니다. 다만 라우터와 양쪽 endpoint가 전부 ECN을 지원해야 하므로, 아직 모든 곳에서 쓰이지는 않습니다.
</div>

## 그런데 왜 알고리즘이 여러 개인가

방금 설명한 건 가장 고전적인 방식(Reno 계열)의 큰 그림입니다. 실제로는 Reno, Cubic, BBR처럼 여러 혼잡제어 알고리즘이 있고, 리눅스는 기본값으로 Cubic을 씁니다. "유실이 발생하면 줄인다"는 방식 자체에 대한 다른 접근(예: 유실이 아니라 지연 시간 변화로 혼잡을 미리 감지하는 방식)도 있습니다. 각 알고리즘이 정확히 뭐가 다른지는 심화편에서 실제 커널 코드를 보면서 다룹니다.

## 다음 글

지금까지 기본 단계에서 연결을 맺고(3편), 데이터를 흐름제어하며 보내고(4편), 유실을 복구하고(5편), 혼잡을 피하는(6편) 법을 봤습니다. 다음 글은 기본 단계의 마지막입니다 — 연결을 어떻게 끝내는지, 4-way handshake와 TIME_WAIT을 다룹니다.
