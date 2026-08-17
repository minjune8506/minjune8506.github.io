---
layout: post
title: "strace로 소켓 syscall 흐름 추적하기"
date: 2026-11-13 09:00:00 +0900
categories: [TCP]
tags: [뇌절, TCP, 심화, OS, syscall]
series: TCP
tier: 심화
mermaid: false
---

[8~13편]({% link _posts/2026-11-06-tcp-13-congestion-control-kernel.md %})에서 커널 내부(상태머신, 재전송, 혼잡제어)를 봤습니다. 이번 글은 시선을 한 단계 올립니다 — 애플리케이션이 부르는 `socket()`, `connect()` 같은 함수가 커널의 그 메커니즘들과 정확히 어느 지점에서 만나는지, syscall 레벨에서 봅니다.

**미리 밝혀둘 게 있습니다.** 이 글을 쓰는 환경은 macOS라서 `strace`(리눅스 전용)도, `dtruss`(SIP 때문에 막혀 있음)도 이 자리에서 직접 실행해서 보여드릴 수 없었습니다. 아래 트레이스는 제가 즉석에서 지어낸 게 아니라 리눅스 syscall의 실제 시그니처와 동작을 기준으로 정확하게 구성한 예시입니다 — 여러분의 리눅스 환경에서 `strace -f -e trace=network ./a.out`으로 직접 찍어보면 아래와 동일한 흐름을 볼 수 있습니다.

## 서버 쪽: socket → bind → listen → accept

```
socket(AF_INET, SOCK_STREAM, 0)                = 3
setsockopt(3, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0
bind(3, {sa_family=AF_INET, sin_port=htons(443),
         sin_addr=inet_addr("0.0.0.0")}, 16)    = 0
listen(3, 128)                                  = 0
accept4(3, {sa_family=AF_INET, sin_port=htons(51000),
            sin_addr=inet_addr("203.0.113.50")}, [16], 0) = 4
```

한 줄씩 지금까지 배운 내용과 연결됩니다.

- `socket()`: 커널에 소켓 자료구조(11편에서 본 `tcp_sock`)를 하나 만들어달라고 요청합니다. 아직 IP도 포트도 없는 상태입니다.
- `bind()`: 2편에서 본 로컬 IP·포트를 이 소켓에 붙입니다. `sin_port=443`이 딱 그 자리입니다.
- `listen()`: 소켓을 `LISTEN` 상태로 바꿉니다(11편에서 본 `tcp_states.h`의 `TCP_LISTEN`). 두 번째 인자 `128`은 3편에서 다룬 backlog 큐 크기입니다.
- `accept4()`: 이 syscall이 호출된 시점부터 3편의 3-way handshake가 시작됩니다. 커널이 SYN을 받고, SYN-ACK를 보내고, 마지막 ACK까지 받아서 연결이 `ESTABLISHED`가 될 때까지, `accept4()`는 그 자리에서 멈춰 있습니다(블로킹 소켓 기준). 연결이 완성되면 그제서야 **새 파일 디스크립터(`4`)**를 들고 리턴합니다 — 원래 듣고 있던 소켓(`3`)은 계속 `LISTEN` 상태로 남아서 다음 연결을 받을 준비를 합니다.

`accept4()`가 반환한 소켓 `4`가 바로 이후 실제 데이터를 주고받는 소켓입니다. `3`과 `4`가 서로 다른 파일 디스크립터라는 게 중요합니다 — 리스닝 소켓 하나로 수천 개의 연결을 받아도, 각 연결은 `accept()`할 때마다 별도의 fd를 받습니다.

## 클라이언트 쪽: socket → connect

```
socket(AF_INET, SOCK_STREAM, 0)                = 3
connect(3, {sa_family=AF_INET, sin_port=htons(443),
            sin_addr=inet_addr("203.0.113.10")}, 16) = 0
```

클라이언트는 `bind()`도 `listen()`도 안 씁니다. `connect()` 하나가 3편의 handshake 전체(SYN 보내기 → SYN-ACK 받기 → ACK 보내기)를 이 한 줄 안에서 다 처리합니다. 이 syscall도 handshake가 완전히 끝나야(즉 `ESTABLISHED`가 돼야) 리턴합니다. 로컬 포트는 어디 갔을까요 — `bind()`를 안 불렀으니, 커널이 2편에서 본 dynamic 포트 범위에서 알아서 하나 골라 씁니다.

## 블로킹이 아니라면 — EAGAIN

지금까지는 전부 블로킹(blocking) 소켓 기준이었습니다. 소켓을 `O_NONBLOCK`으로 만들면 얘기가 달라집니다.

```
connect(3, {...}, 16)                    = -1 EINPROGRESS (Operation now in progress)
...
read(4, ...)                             = -1 EAGAIN (Resource temporarily unavailable)
```

논블로킹 소켓에서 `connect()`는 handshake가 끝날 때까지 기다리지 않고 즉시 `EINPROGRESS`를 반환하며 돌아옵니다 — "지금 진행 중이니 나중에 다시 확인해"라는 뜻입니다. 마찬가지로 아직 도착한 데이터가 없는데 `read()`를 부르면, 멈춰서 기다리는 대신 즉시 `EAGAIN`을 반환합니다. 그럼 "언제 다시 확인해야 하는지"는 어떻게 알까요 — 그게 다음 섹션입니다.

## epoll — "언제 확인하면 되는지"를 커널이 알려준다

논블로킹 소켓 수천 개를 매번 순서대로 `read()`해서 EAGAIN인지 확인하는 건 낭비입니다(이게 이른바 C10K 문제의 출발점입니다). `epoll`은 "준비된 것만" 알려달라고 커널에 등록하는 방식입니다.

```
epoll_create1(0)                          = 5
epoll_ctl(5, EPOLL_CTL_ADD, 4, {events=EPOLLIN, data={fd=4}}) = 0
epoll_wait(5, [{events=EPOLLIN, data={fd=4}}], 10, -1) = 1
read(4, "GET / HTTP/1.1\r\n...", 4096)    = 42
```

`epoll_create1()`로 감시 목록을 하나 만들고, `epoll_ctl()`로 "이 소켓(`4`)에 읽을 데이터가 생기면(`EPOLLIN`) 알려줘"라고 등록합니다. `epoll_wait()`는 등록된 소켓 중 실제로 준비된 게 생길 때까지 블로킹하고, 준비되면 그 목록만 돌려줍니다. 수천 개를 등록해놔도 `epoll_wait()`는 딱 준비된 것만 리턴하므로, 나머지를 일일이 확인할 필요가 없습니다. 이 자료구조가 내부적으로 정확히 어떻게 구현돼 있는지(레드블랙 트리로 등록 목록을 관리하고, 준비 목록은 별도 연결 리스트로 관리한다는 것)는 다음 글에서 커널 소스로 봅니다.

## 정리 — 지금까지 배운 것과의 대응

| syscall | 관련 있는 이전 글 |
|---|---|
| `bind()` | 2편 — 로컬 IP·포트 |
| `listen()` | 3편 — backlog 큐 |
| `accept()` / `connect()` | 3편 — 3-way handshake (이 호출 안에서 전부 일어남) |
| `read()`/`write()`의 반환 바이트 수 | 4편 — 흐름제어로 조절된 양만큼만 오간다 |
| `EAGAIN` | 논블로킹 소켓의 기본 신호 |
| `epoll_wait()` | 다음 글 — 내부 구현 |

## 다음 글

이번 글은 애플리케이션이 커널에게 "뭘 해달라"고 요청하는 쪽이었습니다. 다음 글에서는 그 요청을 받은 커널이 실제 하드웨어(NIC)와 어디서 일을 나눠 하는지, 그리고 방금 미뤄둔 epoll의 내부 구조를 봅니다. 심화 단계의 마지막 글입니다.
