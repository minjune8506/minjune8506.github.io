---
layout: post
title: "12. 소켓 버퍼·재전송 큐·RTO 계산의 실제 구현"
date: 2026-10-30 09:00:00 +0900
categories: [CS]
tags: [뇌절, TCP, 심화, 커널, Linux]
series: TCP
tier: 심화
mermaid: false
---

[지난 글]({% link _posts/2026-10-23-tcp-11-linux-state-machine.md %})에서 상태머신을 코드로 확인했습니다. 이번엔 5편에서 "일정 시간(RTO) 기다렸다가 재전송한다"고 뭉뚱그렸던 그 시간을 정확히 어떻게 계산하는지 봅니다. 여전히 Linux 커널 v6.6, `net/ipv4/tcp_input.c`·`tcp_timer.c`·`include/net/tcp.h` 기준입니다.

## 재전송 큐는 사실 트리다

5편에서 "재전송 큐"라고만 불렀는데, 실제 자료구조는 뭘까요. `include/net/tcp.h`에 이런 헬퍼들이 있습니다.

```c
static inline struct sk_buff *tcp_rtx_queue_head(const struct sock *sk)
{
	return skb_rb_first(&sk->tcp_rtx_queue);
}
...
static inline void tcp_rtx_queue_unlink(struct sk_buff *skb, struct sock *sk)
{
	rb_erase(&skb->rbnode, &sk->tcp_rtx_queue);
}
```

`rb_erase`, `skb_rb_first` — 이름에서 보이듯 `tcp_rtx_queue`는 단순 연결 리스트가 아니라 **red-black tree**입니다. 아직 ACK 안 된 세그먼트(`sk_buff` 하나하나)들이 시퀀스 넘버 순서로 트리에 꽂혀 있습니다.

왜 하필 트리일까요 — 5편에서 다룬 SACK가 이유입니다. SACK로 "1400~1600 구간은 이미 받았다"는 정보가 오면, 리스트를 처음부터 훑는 대신 트리에서 해당 구간만 빠르게 찾아 제거해야 하니까요.

## RTT를 재는 코드 — Jacobson의 그 알고리즘

RTO를 정하려면 먼저 RTT(왕복 시간)를 알아야 합니다. `tcp_rtt_estimator()`(`tcp_input.c:845`) 맨 위 주석부터 흥미롭습니다.

```c
/*	The following amusing code comes from Jacobson's
 *	article in SIGCOMM '88.  Note that rtt and mdev
 *	are scaled versions of rtt and mean deviation.
 *	...
 */
```

Van Jacobson이 1988년에 발표한 그 알고리즘이 지금도 그대로 커널에 살아있습니다. 핵심 업데이트 로직:

```c
if (srtt != 0) {
	m -= (srtt >> 3);	/* m is now error in rtt est */
	srtt += m;		/* rtt = 7/8 rtt + 1/8 new */
	...
	m -= (tp->mdev_us >> 2);   /* similar update on mdev */
	...
	tp->mdev_us += m;		/* mdev = 3/4 mdev + 1/4 new */
```

이동평균(EWMA)입니다. `srtt = 7/8 * srtt + 1/8 * 새_측정값`, `mdev = 3/4 * mdev + 1/4 * 새_오차`.

최신 값에 각각 1/8, 1/4의 가중치를 주고 나머지는 예전 값을 유지합니다. RFC 6298이 권장하는 계수와 정확히 일치합니다.

> "SRTT <- (1 - alpha) \* SRTT + alpha \* R'"
> "RTTVAR <- (1 - beta) \* RTTVAR + beta \* &#124;SRTT - R'&#124;"
> (alpha=1/8, beta=1/4)

## RTO 계산 — "4×"는 어디 갔지?

여기까지는 예상대로입니다. 그런데 실제로 RTO를 계산하는 함수를 보면 뭔가 이상합니다.

```c
static inline u32 __tcp_set_rto(const struct tcp_sock *tp)
{
	return usecs_to_jiffies((tp->srtt_us >> 3) + tp->rttvar_us);
}
```

`srtt_us >> 3`(8로 나눠서 스케일을 원래대로 되돌림) `+ rttvar_us`. 그냥 SRTT + RTTVAR입니다.

그런데 RFC 6298의 공식은 이렇습니다.

> "RTO <- SRTT + max (G, K\*RTTVAR)" (K = 4)

분명히 **4를 곱하라**고 돼 있는데, 커널 코드 어디에도 `* 4`가 안 보입니다. 버그일까요?

초기화 코드를 보면 답이 나옵니다.

```c
} else {
	/* no previous measure. */
	srtt = m << 3;		/* take the measured time to be rtt */
	tp->mdev_us = m << 1;	/* make sure rto = 3*rtt */
	tp->rttvar_us = max(tp->mdev_us, tcp_rto_min_us(sk));
```

첫 RTT 측정값이 `m`일 때, `mdev_us = m << 1`, 즉 `2*m`으로 초기화하면서 주석이 **"make sure rto = 3*rtt"**라고 못박아 둡니다.

실제로 계산해보면: `srtt_us>>3 = m`, `rttvar_us = 2m`이니 `__tcp_set_rto = m + 2m = 3m`. RFC 6298의 첫 샘플 규칙과 비교해보면:

> "SRTT <- R", "RTTVAR <- R/2", "RTO <- SRTT + max(G, K\*RTTVAR)"

RFC식대로 계산하면 `RTO = R + 4*(R/2) = R + 2R = 3R`. **똑같습니다.**

커널의 `rttvar_us`는 RFC가 말하는 "RTTVAR" 그 자체가 아니라, **이미 4를 곱한 값을 처음부터 그 상태로 유지**하고 있던 겁니다. 그러니 최종 계산에서 다시 4를 곱할 필요가 없습니다 — `mdev_us`가 갱신되는 gain 값들(`>>2` 등)도 전부 이 4배 스케일을 유지하도록 설계돼 있습니다.

코드만 보면 "4× 어디 갔지"싶지만, 사실은 처음부터 변수 안에 녹아들어가 있었던 겁니다. 스펙과 구현이 다른 게 아니라, **같은 수학을 다른 스케일로 표현**한 것뿐입니다.

## RTO의 상한·하한·초깃값

`include/net/tcp.h`에 이런 상수들이 있습니다.

```c
#define TCP_RTO_MAX	((unsigned)(120*HZ))
#define TCP_RTO_MIN	((unsigned)(HZ/5))
#define TCP_TIMEOUT_INIT ((unsigned)(1*HZ))	/* RFC6298 2.1 initial RTO value	*/
```

`HZ`는 커널의 초당 타이머 틱 수(보통 1000)로, `120*HZ`는 그냥 120초입니다. 정리하면:

- **최솟값**: 0.2초. 아무리 네트워크가 빨라도 RTO가 이보다 짧아지진 않습니다.
- **최댓값**: 120초(=2분). 어디서 많이 본 숫자입니다 — 8편에서 본 RFC 793의 MSL 값(2분)과 같습니다.
- **초깃값**: 1초. 아직 RTT를 한 번도 측정 못 했을 때(연결 맺기 전, 즉 SYN 보낼 때) 쓰는 값이고, 주석에 `RFC6298 2.1 initial RTO value`라고 정확히 출처가 적혀 있습니다.

## 재전송할 때마다 RTO가 두 배로

타이머가 만료돼서 재전송해도 응답이 없으면 어떻게 될까요? `tcp_retransmit_timer()`(`tcp_timer.c:485`)의 마지막 부분입니다.

```c
} else if (sk->sk_state != TCP_SYN_SENT ||
	   icsk->icsk_backoff >
	   READ_ONCE(net->ipv4.sysctl_tcp_syn_linear_timeouts)) {
	/* Use normal (exponential) backoff unless linear timeouts are
	 * activated.
	 */
	icsk->icsk_rto = min(icsk->icsk_rto << 1, TCP_RTO_MAX);
}
```

`icsk_rto << 1`은 RTO를 2배로 늘리는 것(비트 시프트 1칸 = ×2)이고, `TCP_RTO_MAX`(120초)를 넘지 않게 잘라냅니다. `TCP_TIMEOUT_INIT`(1초)부터 시작해서 실패할 때마다 두 배씩 늘어나는 걸 그래프로 보면 이렇습니다.

<figure class="post-figure">
<svg viewBox="0 0 640 260" role="img" aria-label="재전송이 실패할 때마다 RTO가 1초, 2초, 4초, 8초, 16초, 32초, 64초로 두 배씩 늘어나다가 120초(TCP_RTO_MAX)에서 멈춘다." xmlns="http://www.w3.org/2000/svg">
  <line x1="40" y1="20" x2="40" y2="220" stroke="currentColor" stroke-width="1.5"/>
  <line x1="40" y1="220" x2="600" y2="220" stroke="currentColor" stroke-width="1.5"/>
  <text x="15" y="120" font-size="12" fill="currentColor" font-family="sans-serif" text-anchor="middle" transform="rotate(-90 15 120)">RTO (초)</text>
  <text x="580" y="240" font-size="12" fill="currentColor" fill-opacity="0.7" font-family="sans-serif" text-anchor="middle">재전송 시도 횟수</text>

  <line x1="40" y1="35" x2="600" y2="35" stroke="var(--tcp-warn)" stroke-width="1" stroke-dasharray="3 3"/>
  <text x="580" y="30" font-size="10" fill="var(--tcp-warn)" font-family="sans-serif" text-anchor="end">TCP_RTO_MAX = 120초</text>

  <rect x="55" y="218" width="40" height="2" fill="currentColor"/>
  <text x="75" y="212" font-size="9" fill="currentColor" font-family="sans-serif" text-anchor="middle">1초</text>
  <rect x="120" y="216" width="40" height="4" fill="currentColor"/>
  <text x="140" y="208" font-size="9" fill="currentColor" font-family="sans-serif" text-anchor="middle">2초</text>
  <rect x="185" y="212" width="40" height="8" fill="currentColor"/>
  <text x="205" y="204" font-size="9" fill="currentColor" font-family="sans-serif" text-anchor="middle">4초</text>
  <rect x="250" y="204" width="40" height="16" fill="currentColor"/>
  <text x="270" y="196" font-size="9" fill="currentColor" font-family="sans-serif" text-anchor="middle">8초</text>
  <rect x="315" y="188" width="40" height="32" fill="currentColor"/>
  <text x="335" y="180" font-size="9" fill="currentColor" font-family="sans-serif" text-anchor="middle">16초</text>
  <rect x="380" y="156" width="40" height="64" fill="currentColor"/>
  <text x="400" y="148" font-size="9" fill="currentColor" font-family="sans-serif" text-anchor="middle">32초</text>
  <rect x="445" y="92" width="40" height="128" fill="currentColor"/>
  <text x="465" y="84" font-size="9" fill="currentColor" font-family="sans-serif" text-anchor="middle">64초</text>
  <rect x="510" y="35" width="40" height="185" fill="var(--tcp-warn)"/>
  <text x="530" y="27" font-size="9" font-weight="600" fill="var(--tcp-warn)" font-family="sans-serif" text-anchor="middle">120초(cap)</text>

  <text x="75" y="234" font-size="9" fill="currentColor" fill-opacity="0.6" font-family="sans-serif" text-anchor="middle">0</text>
  <text x="140" y="234" font-size="9" fill="currentColor" fill-opacity="0.6" font-family="sans-serif" text-anchor="middle">1</text>
  <text x="205" y="234" font-size="9" fill="currentColor" fill-opacity="0.6" font-family="sans-serif" text-anchor="middle">2</text>
  <text x="270" y="234" font-size="9" fill="currentColor" fill-opacity="0.6" font-family="sans-serif" text-anchor="middle">3</text>
  <text x="335" y="234" font-size="9" fill="currentColor" fill-opacity="0.6" font-family="sans-serif" text-anchor="middle">4</text>
  <text x="400" y="234" font-size="9" fill="currentColor" fill-opacity="0.6" font-family="sans-serif" text-anchor="middle">5</text>
  <text x="465" y="234" font-size="9" fill="currentColor" fill-opacity="0.6" font-family="sans-serif" text-anchor="middle">6</text>
  <text x="530" y="234" font-size="9" fill="currentColor" fill-opacity="0.6" font-family="sans-serif" text-anchor="middle">7</text>
</svg>
<figcaption>RTO는 1초에서 시작해 실패할 때마다 두 배씩(icsk_rto &lt;&lt; 1) 늘어나다가 TCP_RTO_MAX(120초)에서 잘린다.</figcaption>
</figure>

7번째 시도쯤 되면 이미 128초가 나오지만 120초로 잘립니다.

이 값 자체가 "혼잡한 네트워크에 대고 미친 듯이 재시도하지 말라"는 안전장치입니다 — 6편에서 본 AIMD와 철학이 비슷합니다. 계속 실패하면 더 기다렸다가 시도하라는 거죠.

## 다음 글

RTO는 재전송 하나를 언제 할지 정하는 값이었습니다. 다음 글에서는 애초에 "얼마나 보낼지"를 정하는 혼잡제어로 돌아가서, Reno·Cubic·BBR이 실제 커널 코드 레벨에서 어떻게 다른지 비교합니다.
