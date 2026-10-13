---
title: "SYN Flooding 공격은 어떻게 서버를 마비시키고, 서버는 어떻게 막나요?"
preview: "TCP 연결을 맺으려는 요청만 계속 보내고 마무리하지 않아도 정상 사용자가 접속하지 못하게 만들 수 있습니다. 서버는 이 공격에서 스스로를 어떻게 지킬 수 있을까요?"
tags: [network, tcp, security, linux]
---

SYN Flooding은 **연결을 완성하지 않는 `SYN` 요청을 대량으로 보내, 서버가 "연결 대기 상태"를 저장하는 공간을 고갈시키는 공격**입니다. 서버는 **SYN cookie**로 이 상태를 저장하지 않고도 연결을 검증해서 막습니다.

**서버가 마비되는 이유**

TCP 연결은 3-way handshake로 맺습니다.

1. 클라이언트가 `SYN`을 보냅니다.
2. 서버가 `SYN-ACK`를 보내고, 이 연결을 **SYN 큐(SYN backlog)** 에 저장해 둡니다.
3. 클라이언트가 `ACK`를 보내면 서버는 이 연결을 **accept 큐**로 옮기고, 애플리케이션이 `accept()` 시스템콜로 꺼내 갑니다.

공격자가 3번의 `ACK`를 보내지 않고 `SYN`만 계속 보내면 SYN 큐에 "ACK를 기다리는 연결"이 쌓입니다. 서버는 응답이 없는 항목을 `SYN-ACK` 재전송을 반복하며 한동안 붙들고 있으므로, 큐가 가득 차면 **정상 사용자의 `SYN`까지 버려집니다.** 출발지 IP를 위조해서 보내는 경우가 많아 특정 IP를 차단하는 것만으로는 막기 어렵습니다.

**방어 방법**

- **백로그 크기 늘리기**: 큐를 키우면 버티는 시간이 늘어날 뿐, 공격이 계속되면 결국 가득 찹니다.
- **SYN cookie**: 큐에 쌓이는 상태 자체를 없애 근본 원인을 해결합니다.
- 그 밖에 방화벽의 요청 속도 제한이나 앞단의 DDoS 방어 서비스를 함께 씁니다.

**SYN cookie의 동작**

서버는 `SYN`을 받아도 큐에 저장하지 않습니다. 대신 `SYN-ACK`의 시퀀스 번호에 연결 정보를 해시한 값을 담아 보냅니다. (개념을 단순화한 설명입니다.)

```text
seq = HMAC(
  출발지 IP, 출발지 포트,
  목적지 IP, 목적지 포트,
  timestamp, 서버만 아는 비밀키)
```

정상 클라이언트는 `ACK`에 `seq + 1`을 담아 돌려보냅니다. 서버는 `ack - 1`이 같은 방식으로 계산한 값과 일치하는지 검증하고, 일치할 때 비로소 연결을 만듭니다. 저장해 둔 것이 없으니 `ACK`를 보내지 않는 공격 요청은 서버에 아무 자원도 남기지 않습니다. `timestamp`가 들어 있어 오래된 `ACK`를 재사용하는 공격도 막습니다.

리눅스에서는 `sysctl`로 활성 여부를 확인합니다.

```bash
$ sysctl net.ipv4.tcp_syncookies
net.ipv4.tcp_syncookies = 1
```

`1`은 SYN 큐가 넘칠 때만 SYN cookie를 쓰고, `2`는 항상 씁니다.

**한계: 일부 TCP 옵션을 잃습니다**

SYN cookie는 `SYN`을 저장하지 않으므로, `SYN`에 담겨 온 Window Scale과 SACK 같은 옵션을 `ACK`가 돌아왔을 때 알 수 없습니다. 리눅스는 이 정보를 Timestamp 옵션의 `TSval`에 실어 보내고 `TSecr`로 돌려받는 방식으로 일부 복원하지만, 클라이언트가 Timestamp를 지원할 때만 가능합니다. 그렇지 않으면 이 옵션들 없이 연결되어 처리량이 떨어질 수 있습니다. 그래서 커널은 SYN cookie를 정상 부하용이 아니라 **큐가 넘칠 때만 쓰는 대체 수단**으로 둡니다(`tcp_syncookies = 1`). 또한 회선 대역폭을 가득 채우는 공격까지는 막지 못합니다.

## 참고

- [RFC 4987 — TCP SYN Flooding Attacks and Common Mitigations](https://datatracker.ietf.org/doc/html/rfc4987)
- [Linux Kernel 문서 — IP Sysctl (`tcp_syncookies`)](https://docs.kernel.org/networking/ip-sysctl.html)
- [D. J. Bernstein — SYN cookies](https://cr.yp.to/syncookies.html)
- [Linux 소스 — net/ipv4/syncookies.c](https://github.com/torvalds/linux/blob/master/net/ipv4/syncookies.c)
