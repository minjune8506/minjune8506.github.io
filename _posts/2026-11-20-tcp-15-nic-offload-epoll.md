---
layout: post
title: "NIC 오프로딩(TSO/GRO)과 epoll 내부 구현"
date: 2026-11-20 09:00:00 +0900
categories: [CS]
tags: [뇌절, TCP, 심화, OS, 하드웨어]
series: TCP
tier: 심화
mermaid: false
---

[지난 글]({% link _posts/2026-11-13-tcp-14-strace-socket-syscalls.md %})에서 애플리케이션과 커널의 경계(syscall)를 봤습니다. 심화 단계 마지막인 이번 글은 두 가지를 다룹니다 — 커널과 하드웨어(NIC)의 경계, 그리고 14편에서 미뤄뒀던 epoll의 내부 구조. 여전히 Linux 커널 v6.6 기준입니다.

## TSO — 세그먼트 자르기를 NIC에 떠넘기기

9편에서 MSS(Maximum Segment Size)를 다뤘습니다. 원칙대로라면 커널은 애플리케이션이 보낸 데이터를 MSS 크기(보통 1460바이트)로 하나하나 잘라서, 세그먼트마다 TCP/IP 헤더를 새로 만들고 체크섬을 계산해야 합니다. 100KB를 보낸다면 이 작업을 70번쯤 반복해야 한다는 뜻입니다.

**TSO(TCP Segmentation Offload)**는 이 자르는 작업 자체를 NIC 하드웨어에 떠넘깁니다. 커널은 최대 64KB짜리 큰 `sk_buff` 하나를 만들고, 실제로 자를 필요 없이 "이걸 이 크기로 잘라줘"라는 메타데이터만 붙여서 드라이버에 넘깁니다. `include/linux/skbuff.h`에 있는 그 메타데이터가 이겁니다.

```c
unsigned short	gso_size;    /* 이 크기로 잘라라 (보통 MSS) */
unsigned short	gso_segs;    /* 대략 몇 조각이 나올지 */
unsigned int	gso_type;    /* SKB_GSO_TCPV4 등 - 어떤 프로토콜로 자를지 */
```

NIC가 TSO를 지원하면(`NETIF_F_TSO` 기능 플래그), 이 64KB 뭉치를 그대로 받아서 하드웨어 안에서 실제 MSS 크기 프레임들로 쪼개고, 프레임마다 헤더 복사와 체크섬 계산까지 하드웨어가 처리합니다. 커널 입장에서는 헤더를 70번 만드는 대신 1번만 만든 셈이라, CPU 사용량이 크게 줄어듭니다.

## GRO — 반대 방향, 받을 때도 합쳐서

**GRO(Generic Receive Offload)**는 반대 방향입니다. 작은 패킷 여러 개가 연달아 도착하면, 커널 네트워크 스택 위쪽(TCP 처리 로직)까지 하나씩 올려보내는 대신, 도착한 시점에 미리 병합해서 큰 덩어리 하나로 올려보냅니다. `net/core/gro.c` 맨 위에 이런 정의가 있습니다.

```c
#define MAX_GRO_SKBS 8
```

한 번에 최대 8개까지 패킷을 모아서 병합합니다. 이름이 "**Generic**"인 이유는 TSO와 다르게 하드웨어 지원이 필수가 아니기 때문입니다 — NIC가 못 해주면 드라이버 바로 위 소프트웨어 계층에서 커널이 대신 병합합니다. 결과적으로 상위 TCP 처리 코드(11편·12편에서 본 그 함수들)가 패킷을 8분의 1만 처리해도 되는 셈입니다.

정리하면: **TSO는 나갈 때 크게 만들어서 하드웨어가 자르게 하고, GRO는 들어올 때 작은 걸 모아서 소프트웨어(혹은 하드웨어)가 합칩니다.** 둘 다 "커널이 패킷 하나하나를 상대하는 횟수를 줄인다"는 같은 목적을 반대 방향에서 달성합니다.

## epoll — 왜 등록된 게 많아도 빠른가

14편에서 미뤄둔 걸 봅니다. epoll이 select/poll과 다른 점은 자료구조 설계에 있습니다. `fs/eventpoll.c`의 `struct eventpoll`:

```c
struct eventpoll {
	...
	/* List of ready file descriptors */
	struct list_head rdllist;
	...
	/* RB tree root used to store monitored fd structs */
	struct rb_root_cached rbr;
	...
};
```

**등록된 fd 전체**는 `rbr`(red-black tree)에 있고, **지금 준비된 fd만** 따로 `rdllist`(연결 리스트)에 있습니다. `epoll_ctl(EPOLL_CTL_ADD, ...)`을 부르면 `ep_rbtree_insert()`가 트리에 O(log n)으로 하나 끼워 넣습니다 — 등록 자체는 여기서 끝입니다.

핵심은 그다음입니다. 소켓에 실제로 데이터가 도착하면, 그 소켓의 네트워크 코드가 (poll 대기 중인 쪽에) **콜백을 직접 호출**합니다.

```c
static int ep_poll_callback(wait_queue_entry_t *wait, unsigned mode, int sync, void *key)
{
	...
	} else if (!ep_is_linked(epi)) {
		/* In the usual case, add event to ready list. */
		if (list_add_tail_lockless(&epi->rdllink, &ep->rdllist))
			ep_pm_stay_awake_rcu(epi);
```

이벤트가 생긴 그 순간, 해당 항목이 `rdllist`에 **바로 추가**됩니다. 그래서 `epoll_wait()`(내부적으로 `ep_poll()`)가 할 일은 아주 단순합니다.

```c
eavail = ep_events_available(ep);   /* rdllist가 비어있나 확인 */
while (1) {
	if (eavail) {
		res = ep_send_events(ep, events, maxevents);  /* 준비된 것만 복사 */
		...
```

등록된 게 10개든 10만 개든, `epoll_wait()`는 **등록 전체를 훑지 않고 `rdllist`만 봅니다.** select/poll은 매번 호출될 때마다 등록된 fd 전체를 순서대로 확인해야 하는 것과 정반대입니다.

<figure class="post-figure">
<svg viewBox="0 0 640 220" role="img" aria-label="select/poll은 호출될 때마다 등록된 fd 전체를 순서대로 확인해야 하지만, epoll은 이벤트가 생긴 fd만 콜백으로 ready list에 미리 등록해두고 epoll_wait는 그 목록만 반환한다." xmlns="http://www.w3.org/2000/svg">
  <text x="161" y="20" font-size="13" font-weight="600" fill="currentColor" font-family="sans-serif" text-anchor="middle">select/poll — 매번 전체를 확인 O(N)</text>
  <rect x="20" y="50" width="42" height="40" fill="none" stroke="currentColor" stroke-width="1.5"/>
  <rect x="68" y="50" width="42" height="40" fill="none" stroke="currentColor" stroke-width="1.5"/>
  <rect x="116" y="50" width="42" height="40" fill="none" stroke="currentColor" stroke-width="1.5"/>
  <rect x="164" y="50" width="42" height="40" fill="none" stroke="var(--tcp-accent)" stroke-width="2.5"/>
  <rect x="212" y="50" width="42" height="40" fill="none" stroke="currentColor" stroke-width="1.5"/>
  <rect x="260" y="50" width="42" height="40" fill="none" stroke="currentColor" stroke-width="1.5"/>
  <line x1="20" y1="105" x2="302" y2="105" stroke="currentColor" stroke-width="1.3"/>
  <path d="M 296,100 L 304,105 L 296,110" fill="none" stroke="currentColor" stroke-width="1.3"/>
  <text x="161" y="122" font-size="10" fill="currentColor" fill-opacity="0.75" font-family="sans-serif" text-anchor="middle">호출될 때마다 순서대로 전부 확인</text>
  <text x="161" y="155" font-size="10" fill="currentColor" fill-opacity="0.7" font-family="sans-serif" text-anchor="middle">등록 개수가 많아지면</text>
  <text x="161" y="170" font-size="10" fill="currentColor" fill-opacity="0.7" font-family="sans-serif" text-anchor="middle">매번 그만큼 느려진다</text>

  <text x="480" y="20" font-size="13" font-weight="600" fill="currentColor" font-family="sans-serif" text-anchor="middle">epoll — 콜백이 미리 push</text>
  <rect x="340" y="50" width="42" height="40" fill="none" stroke="currentColor" stroke-width="1.5"/>
  <rect x="388" y="50" width="42" height="40" fill="none" stroke="currentColor" stroke-width="1.5"/>
  <rect x="436" y="50" width="42" height="40" fill="none" stroke="currentColor" stroke-width="1.5"/>
  <rect x="484" y="50" width="42" height="40" fill="none" stroke="var(--tcp-accent)" stroke-width="2.5"/>
  <rect x="532" y="50" width="42" height="40" fill="none" stroke="currentColor" stroke-width="1.5"/>
  <rect x="580" y="50" width="42" height="40" fill="none" stroke="currentColor" stroke-width="1.5"/>
  <text x="460" y="45" font-size="9" fill="currentColor" fill-opacity="0.6" font-family="sans-serif" text-anchor="middle">rbr (등록 전체, rbtree)</text>

  <line x1="505" y1="90" x2="490" y2="128" stroke="var(--tcp-accent)" stroke-width="1.5"/>
  <path d="M 483,122 L 489,130 L 497,124" fill="none" stroke="var(--tcp-accent)" stroke-width="1.5"/>
  <text x="530" y="105" font-size="9" fill="var(--tcp-accent)" font-family="sans-serif" text-anchor="middle">이벤트 발생 시</text>
  <text x="530" y="117" font-size="9" fill="var(--tcp-accent)" font-family="sans-serif" text-anchor="middle">콜백이 직접 추가</text>

  <rect x="440" y="130" width="100" height="34" fill="none" stroke="var(--tcp-accent)" stroke-width="2"/>
  <text x="490" y="151" font-size="10" fill="var(--tcp-accent)" font-family="sans-serif" text-anchor="middle">rdllist</text>
  <text x="490" y="185" font-size="10" fill="currentColor" fill-opacity="0.75" font-family="sans-serif" text-anchor="middle">epoll_wait()는 이 목록만 반환</text>
</svg>
<figcaption>select/poll은 매번 등록된 전체를 훑지만, epoll은 이벤트가 생긴 항목만 콜백이 미리 ready list에 넣어두고 epoll_wait는 그 목록만 본다.</figcaption>
</figure>

1편에서 예고했던 C10K 문제(연결이 1만 개쯤 되면 서버가 감당을 못 하던 문제)가 정확히 이 구조 차이 때문에 풀립니다. select/poll은 연결이 늘어날수록 매 호출마다 확인해야 할 목록도 같이 늘어나지만, epoll은 등록 개수와 무관하게 "지금 진짜로 준비된 것"만 봅니다.

## 심화 단계를 마치며

8편부터 여기까지, 기본편에서 "이렇게 동작한다"고 설명했던 것들을 RFC 원문(8~10편)과 실제 Linux 커널 소스(11~15편)로 하나씩 검증했습니다. ISN 생성 공식, RTO 계산의 숨은 스케일링, Cubic이 실제로는 절반이 아니라 70%만 줄인다는 것, epoll이 콜백으로 동작한다는 것 — 전부 코드를 직접 열어보지 않았다면 몰랐을 디테일입니다.

## 다음 글

여기서 심화 단계가 끝나고, **뇌절** 단계로 넘어갑니다. 지금까지는 읽고 검증만 했다면, 이제부터는 직접 손으로 만듭니다 — 첫 글은 Wireshark로 실제 handshake 패킷을 캡처해서 지금까지 본 모든 걸 눈으로 대조하는 것부터 시작합니다.
