---
title: "HTTP vs HTTPS, 그리고 SSL 핸드셰이크"
date: 2026-11-16 20:00:00 +0900
categories: [Network, HTTP]
tags: [http, https, tls, ssl, handshake, interview]
description: HTTP와 HTTPS가 무엇이 다른지, HTTPS가 TLS로 무엇을 지키는지, 데이터를 보내기 전에 오가는 TLS 핸드셰이크 메시지를 순서대로 정리했다.
toc: true
lang: ko-KR
ref: http-vs-https
---

면접에서 "HTTP와 HTTPS의 차이는요?"에 "HTTPS는 암호화합니다"라고 답하면 바로 "무엇으로, 언제 암호화하나요?"라는 질문이 이어진다. 이 글은 HTTPS가 HTTP에 무엇을 더한 것인지, 그리고 암호화를 시작하기 전에 오가는 핸드셰이크를 순서대로 정리한다.

> **요약**
> - HTTPS는 HTTP 메시지를 TLS 연결 위에서 보내는 것이다. HTTP 문법(메서드, 헤더, 상태 코드)은 똑같다.
> - HTTP는 요청과 응답을 평문으로 보낸다. 중간 네트워크에서 패킷을 보면 비밀번호와 쿠키가 그대로 읽힌다.
> - TLS는 세 가지를 더한다. 내용 암호화, 변조 감지, 인증서로 서버가 진짜인지 확인하는 것이다.
> - 핸드셰이크에서는 비대칭 키 방식으로 대칭 키를 합의하고 실제 데이터는 그 대칭 키로 암호화한다.
> - TLS 1.2는 핸드셰이크에 2번 왕복(2-RTT), TLS 1.3은 1번 왕복(1-RTT)이 든다.
{: .prompt-tip }

## 먼저 알아둘 단어

| 단어 | 뜻 |
| --- | --- |
| TLS | Transport Layer Security. HTTPS의 암호화를 맡는 프로토콜 |
| SSL | TLS의 이전 이름. 지금은 모든 버전이 금지됐다 |
| 대칭 키 | 암호화와 복호화에 같은 키를 쓰는 방식(예: AES) |
| 비대칭 키 | 공개 키와 개인 키 한 쌍을 쓰는 방식(예: RSA, ECDHE) |
| 인증서 | 이 공개 키가 이 도메인의 것임을 CA가 서명한 파일 |
| CA | 인증서에 서명하는 기관. 브라우저와 OS에 루트 CA 목록이 들어 있다 |
| RTT | 클라이언트와 서버 사이를 한 번 왕복하는 시간 |

## 1. HTTP와 HTTPS의 차이

### 1-1. 한 문장으로

HTTP는 TCP 연결에 HTTP 메시지를 바로 쓴다. HTTPS는 TCP 연결 위에 먼저 TLS 연결을 만들고 그 안에 HTTP 메시지를 쓴다.

```text
HTTP                 HTTPS
+--------+           +--------+
|  HTTP  |           |  HTTP  |
+--------+           +--------+
|  TCP   |           |  TLS   |   <- 암호화, 무결성, 인증
+--------+           +--------+
|   IP   |           |  TCP   |
+--------+           +--------+
                     |   IP   |
                     +--------+
```

### 1-2. 비교 표

| 항목 | HTTP | HTTPS |
| --- | --- | --- |
| URL 스킴 | `http://` | `https://` |
| 기본 포트 | 80 | 443 |
| 전송 내용 | 평문 | TLS로 암호화 |
| 서버 확인 | 하지 않음 | 인증서로 도메인 확인 |
| 변조 감지 | 없음 | 레코드마다 검사 |
| 연결 비용 | TCP 핸드셰이크 | TCP + TLS 핸드셰이크 |

### 1-3. HTTP를 쓰면 실제로 생기는 일

카페 와이파이에서 `http://`로 로그인하면 같은 네트워크에서 패킷을 캡처하는 사람은 아래 요청을 그대로 읽는다.

```text
POST /login HTTP/1.1
Host: example.com
Cookie: SESSION=8f2a...

username=kim&password=1234
```

내용이 바뀔 수도 있다. 중간에 있는 공유기나 프록시가 응답 HTML에 광고 스크립트를 끼워 넣어도 브라우저는 알아채지 못한다. 가짜 서버가 `example.com`인 척 응답해도 확인할 방법이 없다.

### 1-4. HTTPS를 써도 보이는 것

HTTPS에서는 URL 경로, 쿼리 문자열, 헤더, 쿠키, 본문이 모두 암호화된다. 그래도 아래 정보는 중간에서 보인다.

- 접속한 서버의 IP 주소
- 접속한 도메인 이름. 핸드셰이크의 SNI 필드와 DNS 조회에 평문으로 들어간다.
- 주고받은 데이터의 크기와 시각

"HTTPS를 쓰면 어느 사이트에 갔는지도 숨겨진다"는 말은 틀렸다. HTTPS는 그 사이트 안에서 무엇을 했는지만 숨긴다.

## 2. HTTPS가 TLS로 지키는 세 가지

| 목표 | 막는 공격 | 쓰는 기술 |
| --- | --- | --- |
| 기밀성 | 패킷 엿보기 | 대칭 키 암호화(AES-GCM 등) |
| 무결성 | 중간에서 내용 바꾸기 | AEAD 인증 태그 |
| 인증 | 가짜 서버 | 인증서와 서명 |

### 2-1. 왜 대칭 키와 비대칭 키를 섞어 쓰나

대칭 키 암호화는 CPU 연산이 적어서 대용량 데이터를 암호화하는 데 쓴다. 다만 양쪽에 같은 키가 있어야 한다. 키를 평문으로 보내면 엿보는 사람도 키를 얻는다.

비대칭 키 방식은 공개 키를 공개해도 안전하다. 대신 연산량이 대칭 키보다 훨씬 많아서 모든 데이터에 쓰기에는 맞지 않는다.

그래서 TLS는 둘을 나눠 쓴다.

1. 핸드셰이크에서 비대칭 키 방식(ECDHE)으로 양쪽이 같은 비밀 값을 만든다. 이 값은 네트워크에 흐르지 않는다.
2. 그 비밀 값에서 대칭 키를 뽑아 이후 모든 HTTP 데이터를 암호화한다.

### 2-2. 인증서로 서버를 확인하는 방법

브라우저는 서버가 보낸 인증서를 받으면 아래를 확인한다. 하나라도 실패하면 "연결이 비공개로 설정되어 있지 않습니다" 같은 경고 화면을 띄운다.

1. 인증서의 도메인(SAN 항목)이 주소창의 도메인과 같은가
2. 유효 기간 안인가
3. 서명을 따라 올라가면 브라우저나 OS에 들어 있는 루트 CA에 닿는가
4. 서버에 그 인증서의 개인 키가 실제로 있는가. 핸드셰이크 중 서명으로 증명한다.

인증서 파일 자체는 공개 정보라서 누구나 복사할 수 있다. 개인 키가 없으면 핸드셰이크의 서명을 만들 수 없어서 가짜 서버는 4번에서 걸린다.

## 3. SSL/TLS 핸드셰이크

"SSL 핸드셰이크"라는 이름이 아직 많이 쓰인다. 지금 실제로는 TLS 핸드셰이크가 일어난다. SSL 2.0과 3.0은 RFC로 사용이 금지됐다. TLS 1.0과 1.1도 2021년 RFC 8996으로 폐기됐다. 현재 쓰는 버전은 TLS 1.2와 TLS 1.3이다.

핸드셰이크는 TCP 연결이 맺어진 직후, HTTP 요청을 보내기 전에 일어난다. 하는 일은 세 가지다.

- 사용할 TLS 버전과 암호 스위트를 정한다.
- 서버 인증서를 확인한다.
- 대칭 키를 만들 비밀 값을 합의한다.

### 3-1. TLS 1.2 핸드셰이크 (ECDHE 기준)

```text
클라이언트                                   서버
    |                                         |
    | --- ClientHello ----------------------> |  지원 버전, 암호 스위트 목록, 랜덤 값
    |                                         |
    | <-- ServerHello ----------------------- |  고른 버전, 고른 암호 스위트, 랜덤 값
    | <-- Certificate ----------------------- |  서버 인증서 체인
    | <-- ServerKeyExchange ----------------- |  ECDHE 공개 값 + 개인 키 서명
    | <-- ServerHelloDone ------------------- |
    |                                         |   (1 RTT 끝)
    |  인증서 검증, 비밀 값 계산               |
    |                                         |
    | --- ClientKeyExchange ----------------> |  클라이언트 ECDHE 공개 값
    | --- ChangeCipherSpec -----------------> |  "지금부터 암호화한다"
    | --- Finished (암호화) ----------------> |  지금까지 메시지의 해시
    |                                         |
    | <-- ChangeCipherSpec ------------------ |
    | <-- Finished (암호화) ----------------- |
    |                                         |   (2 RTT 끝)
    | === HTTP 요청/응답 (암호화) ==========> |
```

메시지마다 하는 일은 이렇다.

1. **ClientHello**: 클라이언트가 지원하는 TLS 버전, 암호 스위트 목록, 랜덤 값 32바이트를 보낸다. 접속하려는 도메인도 SNI 확장에 담는다. 서버 하나에 도메인이 여러 개 있을 때 어떤 인증서를 줄지 고르는 데 쓴다.
2. **ServerHello**: 서버가 목록에서 버전과 암호 스위트를 하나씩 고르고 자기 랜덤 값을 보낸다.
3. **Certificate**: 서버 인증서와 중간 CA 인증서를 보낸다.
4. **ServerKeyExchange**: 서버가 ECDHE 공개 값을 보낸다. 이 값에 인증서 개인 키로 서명을 붙인다. 2-2절 4번의 "서버에 개인 키가 있다"는 증명이 여기서 일어난다.
5. **ClientKeyExchange**: 클라이언트가 인증서를 검증한 뒤 자기 ECDHE 공개 값을 보낸다. 이제 양쪽은 상대 공개 값과 자기 개인 값으로 같은 비밀 값(pre-master secret)을 계산한다.
6. 양쪽은 pre-master secret과 두 랜덤 값을 섞어 master secret을 만들고 거기서 대칭 키를 뽑는다.
7. **ChangeCipherSpec, Finished**: "이제부터 새 키로 암호화한다"고 알린 뒤 지금까지 주고받은 핸드셰이크 메시지 전체의 해시를 암호화해서 보낸다. 상대가 계산한 해시와 다르면 중간에서 메시지가 바뀐 것이므로 연결을 끊는다.

> 예전 설명에는 "클라이언트가 pre-master secret을 서버 공개 키(RSA)로 암호화해서 보낸다"는 방식이 자주 나온다. 이것은 RSA 키 교환이고 TLS 1.3에서 제거됐다. 서버 개인 키가 나중에 유출되면 녹화해 둔 과거 트래픽을 모두 복호화할 수 있기 때문이다. ECDHE는 연결마다 임시 키를 만들고 버리므로 개인 키가 유출돼도 과거 트래픽은 풀리지 않는다(전방 비밀성, forward secrecy).
{: .prompt-warning }

### 3-2. TLS 1.3 핸드셰이크

TLS 1.3은 클라이언트가 첫 메시지에 키 교환 값을 미리 넣어서 왕복을 하나 줄였다.

```text
클라이언트                                   서버
    |                                         |
    | --- ClientHello + key_share ----------> |  지원 그룹별 ECDHE 공개 값을 미리 보냄
    |                                         |
    | <-- ServerHello + key_share ----------- |  서버 ECDHE 공개 값
    |     (여기부터 암호화)                    |
    | <-- EncryptedExtensions --------------- |
    | <-- Certificate ----------------------- |
    | <-- CertificateVerify ----------------- |  개인 키 서명
    | <-- Finished -------------------------- |
    |                                         |   (1 RTT 끝)
    | --- Finished -------------------------> |
    | === HTTP 요청 (암호화) ===============> |
```

TLS 1.2와 달라진 점은 네 가지다.

| 항목 | TLS 1.2 | TLS 1.3 |
| --- | --- | --- |
| 첫 요청까지 왕복 | 2 RTT | 1 RTT |
| RSA 키 교환 | 허용 | 제거 |
| 인증서 전송 | 평문 | 암호화 |
| 암호 스위트 | 키 교환까지 묶은 조합 | 암호+해시만, 5개 |

서버에서 클라이언트까지 왕복이 100ms라면 TLS 1.3은 첫 HTTP 요청을 100ms 일찍 보낸다.

> TLS 1.3에는 이전에 접속한 서버에 첫 메시지부터 데이터를 실어 보내는 0-RTT 모드도 있다. 대신 공격자가 그 메시지를 녹화해서 다시 보내는 재전송 공격을 막지 못한다. 그래서 0-RTT로는 GET처럼 여러 번 실행해도 결과가 같은 요청만 보내야 한다.
{: .prompt-info }

### 3-3. 직접 확인하기

`curl -v`로 접속하면 핸드셰이크 과정과 협상된 버전, 암호 스위트가 출력된다.

```bash
curl -v https://example.com -o /dev/null
```

인증서 체인까지 보고 싶으면 `openssl`을 쓴다.

```bash
openssl s_client -connect example.com:443 -servername example.com
```

## 4. 자주 나오는 꼬리 질문

**HTTPS로 바꾸면 느려지나요?**

새 연결마다 핸드셰이크 왕복이 TLS 1.2는 2번, TLS 1.3은 1번 더 든다. keep-alive로 연결 하나에 요청을 여러 번 보내면 핸드셰이크는 처음 한 번만 한다. 데이터 암호화는 AES 하드웨어 명령(AES-NI)이 있는 CPU에서 처리하므로 체감 차이를 만드는 것은 대부분 왕복 횟수다.

**http로 들어오면 https로 리다이렉트하면 충분한가요?**

첫 요청이 평문 HTTP로 나가기 때문에 그 순간을 가로채면 리다이렉트 자체를 막을 수 있다. 응답에 `Strict-Transport-Security` 헤더(HSTS)를 붙이면 브라우저는 지정한 기간 동안 그 도메인에 처음부터 HTTPS로만 접속한다.

```text
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

**TLS는 어디서 끝내나요?**

Nginx 같은 웹 서버가 TLS를 풀고 뒤쪽 애플리케이션 서버에는 HTTP로 넘기는 구성이 많다. 인증서를 웹 서버 한 곳에만 설치하면 된다. 이 구성은 [웹 서버와 WAS 글](/posts/webserver-vs-was/)에 정리했다.

## 정리

- HTTP는 요청과 응답을 평문으로 보낸다. HTTPS는 같은 HTTP 메시지를 TLS 연결 안에서 보낸다. 기본 포트는 80과 443이다.
- TLS는 내용을 암호화하고, 바뀐 데이터를 감지하고, 인증서로 서버가 그 도메인의 주인인지 확인한다.
- HTTPS여도 접속한 IP와 도메인 이름은 중간에서 보인다. 경로, 헤더, 쿠키, 본문은 보이지 않는다.
- 핸드셰이크는 ECDHE로 비밀 값을 합의하고 거기서 뽑은 대칭 키로 이후 데이터를 암호화한다.
- TLS 1.2는 ClientHello부터 Finished까지 2 RTT, TLS 1.3은 ClientHello에 키 교환 값을 미리 넣어 1 RTT가 든다.
- SSL은 모든 버전이 금지됐다. "SSL 핸드셰이크"라고 불러도 실제로는 TLS 1.2나 1.3 핸드셰이크다.

## 참고 문서

- [RFC 9110: HTTP Semantics (http, https URI 스킴과 기본 포트)](https://www.rfc-editor.org/rfc/rfc9110)
- [RFC 8446: The Transport Layer Security (TLS) Protocol Version 1.3](https://www.rfc-editor.org/rfc/rfc8446)
- [RFC 5246: The Transport Layer Security (TLS) Protocol Version 1.2](https://www.rfc-editor.org/rfc/rfc5246)
- [RFC 7568: Deprecating Secure Sockets Layer Version 3.0](https://www.rfc-editor.org/rfc/rfc7568)
- [RFC 8996: Deprecating TLS 1.0 and TLS 1.1](https://www.rfc-editor.org/rfc/rfc8996)
- [RFC 6797: HTTP Strict Transport Security (HSTS)](https://www.rfc-editor.org/rfc/rfc6797)
- [MDN: Transport Layer Security](https://developer.mozilla.org/en-US/docs/Web/Security/Transport_Layer_Security)
