---
title: "TLS 1.3에서 RSA 키 교환이 삭제된 이유는 무엇인가요?"
preview: "TLS 1.2까지는 서버 공개키로 비밀을 암호화해 보내는 방식이 널리 쓰였습니다. 지금 당장은 아무도 풀 수 없는데, 왜 1.3에서 아예 빠졌을까요?"
tags: [network, tls, security, cryptography]
---

RSA 키 교환은 **세션 키의 재료를 서버 공개키로 암호화해서 망으로 보내는 방식**이라, 서버 개인키가 나중에 유출되면 **과거에 녹화된 트래픽까지 전부 복호화**됩니다. 이 성질을 막는 **Forward Secrecy**가 없어서 TLS 1.3에서 삭제됐고, 대신 연결마다 임시 키로 공유 비밀을 계산하는 **ECDHE**만 남았습니다. 다만 DH 계열은 상대가 누군지 확인하지 못하므로, 서버는 인증서 개인키로 **서명**해서 자신을 증명합니다.

**RSA 키 교환은 어떻게 동작하나요**

TLS 통신 자체는 빠른 대칭키로 암호화합니다. 그래서 handshake의 핵심은 "처음 보는 상대와, 도청당하는 망 위에서, 어떻게 같은 대칭키를 갖느냐"입니다. TLS는 세션 키 자체를 보내지 않고, **pre-master secret**이라는 공유 비밀을 맞춘 뒤 양쪽이 각자 세션 키를 파생합니다.

RSA 키 교환(TLS 1.2)은 이 공유 비밀을 **운반**합니다.

1. `ClientHello`: 클라이언트가 지원하는 cipher suite와 `client random`(32바이트)을 보냅니다.
2. `ServerHello`, `Certificate`: 서버가 cipher suite를 고르고 `server random`과 인증서를 보냅니다.
3. `ClientKeyExchange`: 클라이언트가 48바이트 pre-master secret을 만들어 **인증서 공개키로 암호화**해서 보냅니다.
4. 서버는 **개인키로 복호화**합니다. 양쪽은 같은 공식으로 세션 키를 만듭니다.
5. `Finished`: handshake 전체의 해시를 세션 키로 암호화해 교환합니다. 개인키가 없는 가짜 서버는 pre-master secret을 풀지 못하므로 여기서 걸러집니다.

```text
master secret = PRF(pre-master secret, "master secret", client random + server random)
세션 키들     = PRF(master secret, "key expansion", server random + client random)
               → 클라이언트용·서버용 암호화 키, IV 등으로 잘라 사용
```

흔히 "세션 키"라고 하나로 부르지만, 실제로는 송신 방향마다 다른 대칭키를 씁니다. 클라이언트가 보낼 때는 클라이언트용 키로 암호화하고 서버는 같은 키로 복호화하며, 반대 방향은 서버용 키를 씁니다. 한쪽이 보낸 암호문을 그대로 되돌려 보내는 공격을 막기 위해서입니다.

**무엇이 문제인가요**

위 공식에서 비밀은 **pre-master secret 하나뿐**입니다. PRF는 RFC에 공개된 함수이고, 두 random은 handshake에서 평문으로 지나갑니다. "세션 키는 양쪽이 각자 만든다"는 말은 세션 키를 망에 직접 보내지 않는다는 뜻일 뿐, pre-master secret만 손에 넣으면 누구든 같은 세션 키를 만들 수 있습니다.

공격자가 지금은 복호화하지 못하더라도 트래픽을 **녹화만** 해둔다고 가정해 봅시다. 녹화본에는 다음이 남습니다.

- `client random`, `server random` (평문)
- `Enc(서버 공개키, pre-master secret)`
- 그 뒤 암호화된 애플리케이션 데이터

몇 년 뒤 서버 침해, 백업 유출, Heartbleed 같은 메모리 노출 버그로 서버 개인키가 새어 나가면 이렇게 됩니다.

```text
훔친 개인키로 pre-master secret 복호화
→ 녹화된 두 random과 함께 세션 키 재계산
→ 그 개인키를 쓰던 기간의 녹화본 전부 복호화
```

인증서 개인키는 보통 몇 달에서 몇 년씩 같은 것을 씁니다. RSA 키 교환에서는 그 **개인키 하나가 과거 모든 연결을 여는 열쇠**가 됩니다.

**ECDHE는 무엇이 다른가요**

ECDHE(Elliptic Curve Diffie-Hellman Ephemeral)는 공유 비밀을 운반하지 않고 **합의**합니다.

1. 클라이언트와 서버가 이번 연결에만 쓸 임시 비밀 `a`, `b`를 각자 만듭니다.
2. 여기서 계산한 공개값 `A`, `B`만 교환합니다.
3. 클라이언트는 `a`와 `B`로, 서버는 `b`와 `A`로 같은 공유 비밀 `S`를 계산합니다. `S`가 pre-master secret 역할을 합니다.
4. handshake가 끝나면 `a`, `b`는 버립니다.

공격자의 녹화본에는 `A`, `B`만 남습니다. 공개값만으로 `S`를 계산하는 효율적인 방법은 알려져 있지 않고, `S`를 만들려면 이미 버려진 `a`나 `b`가 필요합니다. 나중에 서버 개인키가 유출돼도 **녹화본에서 꺼낼 비밀이 없습니다.** 이것이 Forward Secrecy(전방향 보안)입니다.

| | RSA 키 교환 | ECDHE |
|---|---|---|
| 공유 비밀을 맞추는 방법 | 클라이언트가 만들어 **운반** | 양쪽이 각자 **계산** |
| 비밀이 망을 지나가나 | 암호화된 채로 지나감 | 공개값만 지나감 |
| 인증서 개인키의 용도 | pre-master secret **복호화** | 서버 공개값에 **서명** |
| 개인키 유출 시 과거 트래픽 | 복호화됨 | 안전 |

`a`, `b`를 연결마다 새로 만드는 것(Ephemeral)이 핵심입니다. 같은 DH라도 임시 비밀을 오래 재사용하면, 그 값이 유출되는 순간 그 기간의 트래픽이 풀립니다. TLS 1.3이 RSA와 함께 **정적(static) DH**도 삭제한 이유입니다.

**ECDHE만으로는 왜 부족한가요**

DH는 공개값을 주고받을 뿐, **그 공개값을 누가 보냈는지 증명하지 못합니다.** 중간자가 `A`, `B`를 자기 공개값으로 바꿔치기하면 클라이언트와 서버는 각각 공격자와 공유 비밀을 만들고, 공격자는 양쪽 트래픽을 풀어서 읽은 뒤 다시 암호화해 전달할 수 있습니다.

그래서 TLS는 키 교환과 인증을 나눠 맡깁니다.

- **키 교환**: ECDHE 임시 키쌍. 연결마다 새로 만들고 버립니다.
- **인증**: 인증서 키쌍. 서버는 handshake 내용(서버 공개값 `B` 포함)에 **인증서 개인키로 서명**합니다. TLS 1.2에서는 `ServerKeyExchange`, TLS 1.3에서는 `CertificateVerify`에 담깁니다.

클라이언트는 OS에 내장된 Root CA부터 인증서 체인을 따라 내려와 인증서 공개키를 믿고, 그 공개키로 서명을 검증합니다. `B`가 바뀌면 서명이 맞지 않아 handshake가 실패합니다. 인증서 개인키는 서명에만 쓰이므로, 유출되면 그 뒤로 서버를 사칭할 수는 있어도 과거 트래픽을 복호화하는 데는 쓸 수 없습니다.

<!-- **덤: 1 RTT가 줄어든 이유와도 연결됩니다**

TLS 1.2에서는 클라이언트가 서버의 선택을 들은 뒤에야 키 재료를 보낼 수 있어 handshake에 2 RTT가 들었습니다. TLS 1.3은 키 교환을 (EC)DHE 계열로 좁혔기 때문에, 클라이언트가 `ClientHello`에 서버가 고를 만한 그룹의 공개값(`key_share`)을 미리 넣어 보낼 수 있습니다. 서버는 첫 응답에서 바로 공유 비밀을 계산하므로 handshake가 1 RTT로 줄었습니다. 서버가 그 그룹을 지원하지 않으면 `HelloRetryRequest`로 1 RTT가 추가됩니다. -->

## 참고

- [RFC 8446 — The Transport Layer Security (TLS) Protocol Version 1.3](https://datatracker.ietf.org/doc/html/rfc8446) (1.2절: static RSA·DH 삭제)
- [RFC 5246 — The Transport Layer Security (TLS) Protocol Version 1.2](https://datatracker.ietf.org/doc/html/rfc5246) (7.4.7절 ClientKeyExchange, 8.1절 master secret 계산)
- [RFC 8422 — ECC Cipher Suites for TLS 1.2 and Earlier](https://datatracker.ietf.org/doc/html/rfc8422)
