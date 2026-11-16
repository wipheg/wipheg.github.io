---
title: "HTTP vs HTTPS, and the SSL Handshake"
date: 2026-11-16 20:00:00 +0900
categories: [Network, HTTP]
tags: [http, https, tls, ssl, handshake, interview]
description: What differs between HTTP and HTTPS, what HTTPS protects with TLS, and the TLS handshake messages exchanged before any data is sent, in order.
toc: true
lang: en
ref: http-vs-https
hidden: true
---

In an interview, if you answer "What is the difference between HTTP and HTTPS?" with "HTTPS encrypts the traffic," the next question is "Encrypts it with what, and when?" This post covers what HTTPS adds on top of HTTP, and the handshake that runs before encryption starts, step by step.

> **Summary**
> - HTTPS is HTTP messages sent over a TLS connection. The HTTP syntax (methods, headers, status codes) is the same.
> - HTTP sends requests and responses in plain text. Anyone capturing packets on the network in between can read passwords and cookies as they are.
> - TLS adds three things: encryption of the content, tamper detection, and checking that the server is genuine with a certificate.
> - The handshake uses asymmetric keys to agree on a symmetric key, and the actual data is encrypted with that symmetric key.
> - The handshake takes two round trips (2-RTT) in TLS 1.2 and one round trip (1-RTT) in TLS 1.3.
{: .prompt-tip }

## Terms to Know First

| Term | Meaning |
| --- | --- |
| TLS | Transport Layer Security. The protocol that does the encryption in HTTPS |
| SSL | The former name of TLS. Every version is now prohibited |
| Symmetric key | The same key encrypts and decrypts (e.g. AES) |
| Asymmetric key | A public/private key pair (e.g. RSA, ECDHE) |
| Certificate | A file in which a CA signs that this public key belongs to this domain |
| CA | An authority that signs certificates. Browsers and OSes ship a list of root CAs |
| RTT | The time for one round trip between client and server |

## 1. The Difference Between HTTP and HTTPS

### 1-1. In One Sentence

HTTP writes HTTP messages directly onto a TCP connection. HTTPS first builds a TLS connection on top of TCP and writes the HTTP messages inside it.

```text
HTTP                 HTTPS
+--------+           +--------+
|  HTTP  |           |  HTTP  |
+--------+           +--------+
|  TCP   |           |  TLS   |   <- encryption, integrity, authentication
+--------+           +--------+
|   IP   |           |  TCP   |
+--------+           +--------+
                     |   IP   |
                     +--------+
```

### 1-2. Comparison Table

| Item | HTTP | HTTPS |
| --- | --- | --- |
| URL scheme | `http://` | `https://` |
| Default port | 80 | 443 |
| Payload | Plain text | Encrypted by TLS |
| Server check | None | Domain checked by certificate |
| Tamper detection | None | Checked per record |
| Connection cost | TCP handshake | TCP + TLS handshake |

### 1-3. What Actually Happens Over HTTP

If you log in over `http://` on café Wi-Fi, anyone capturing packets on the same network reads the request below as is.

```text
POST /login HTTP/1.1
Host: example.com
Cookie: SESSION=8f2a...

username=kim&password=1234
```

The content can also be changed. If a router or proxy in between injects an ad script into the response HTML, the browser does not notice. If a fake server responds pretending to be `example.com`, there is no way to tell.

### 1-4. What Is Still Visible Over HTTPS

With HTTPS, the URL path, query string, headers, cookies, and body are all encrypted. The following is still visible in between.

- The IP address of the server you connected to
- The domain name you connected to. It goes out in plain text in the handshake's SNI field and in the DNS lookup.
- The size and timing of the data exchanged

So "HTTPS hides which site you visited" is wrong. HTTPS only hides what you did on that site.

## 2. Three Things HTTPS Protects with TLS

| Goal | Attack it stops | Technique |
| --- | --- | --- |
| Confidentiality | Sniffing packets | Symmetric encryption (AES-GCM, etc.) |
| Integrity | Altering content in transit | AEAD authentication tag |
| Authentication | Fake server | Certificate and signature |

### 2-1. Why Mix Symmetric and Asymmetric Keys

Symmetric encryption needs little CPU work, so it is used to encrypt large amounts of data. But both sides need the same key. If the key is sent in plain text, an eavesdropper gets the key too.

With asymmetric keys, the public key is safe to publish. But it costs far more computation than symmetric encryption, so it does not fit for encrypting all the data.

So TLS splits the work.

1. During the handshake, both sides derive the same secret value with an asymmetric method (ECDHE). This value never travels over the network.
2. A symmetric key is derived from that secret and encrypts all subsequent HTTP data.

### 2-2. How a Certificate Verifies the Server

When the browser receives the server's certificate, it checks the following. If any check fails, it shows a warning page such as "Your connection is not private."

1. Does the certificate's domain (SAN entry) match the domain in the address bar?
2. Is it within its validity period?
3. Following the signatures upward, does it reach a root CA built into the browser or OS?
4. Does the server actually have the certificate's private key? This is proven with a signature during the handshake.

The certificate file itself is public, so anyone can copy it. Without the private key, a fake server cannot produce the handshake signature, so it fails at step 4.

## 3. The SSL/TLS Handshake

The name "SSL handshake" is still widely used. What actually runs today is the TLS handshake. SSL 2.0 and 3.0 are prohibited by RFC. TLS 1.0 and 1.1 were deprecated in 2021 by RFC 8996. The versions in use now are TLS 1.2 and TLS 1.3.

The handshake runs right after the TCP connection is established and before any HTTP request is sent. It does three things.

- Decide the TLS version and cipher suite to use.
- Verify the server certificate.
- Agree on the secret value used to make the symmetric key.

### 3-1. The TLS 1.2 Handshake (with ECDHE)

```text
Client                                       Server
    |                                         |
    | --- ClientHello ----------------------> |  supported versions, cipher suites, random
    |                                         |
    | <-- ServerHello ----------------------- |  chosen version, chosen suite, random
    | <-- Certificate ----------------------- |  server certificate chain
    | <-- ServerKeyExchange ----------------- |  ECDHE public value + signature
    | <-- ServerHelloDone ------------------- |
    |                                         |   (end of 1 RTT)
    |  verify cert, compute secret            |
    |                                         |
    | --- ClientKeyExchange ----------------> |  client ECDHE public value
    | --- ChangeCipherSpec -----------------> |  "encrypting from now on"
    | --- Finished (encrypted) -------------> |  hash of all messages so far
    |                                         |
    | <-- ChangeCipherSpec ------------------ |
    | <-- Finished (encrypted) -------------- |
    |                                         |   (end of 2 RTT)
    | === HTTP request/response (encrypted) > |
```

What each message does:

1. **ClientHello**: The client sends its supported TLS versions, a list of cipher suites, and a 32-byte random value. It also puts the target domain in the SNI extension. When one server hosts several domains, the server uses it to pick which certificate to send.
2. **ServerHello**: The server picks one version and one cipher suite from the lists and sends its own random value.
3. **Certificate**: The server sends its certificate and the intermediate CA certificates.
4. **ServerKeyExchange**: The server sends its ECDHE public value, signed with the certificate's private key. The proof in step 4 of section 2-2, "the server has the private key," happens here.
5. **ClientKeyExchange**: After verifying the certificate, the client sends its own ECDHE public value. Now both sides compute the same secret (pre-master secret) from the other side's public value and their own private value.
6. Both sides mix the pre-master secret with the two random values to make the master secret, and derive symmetric keys from it.
7. **ChangeCipherSpec, Finished**: Each side announces "encrypting with the new keys from now on," then sends an encrypted hash of all handshake messages exchanged so far. If it differs from the hash the other side computed, a message was altered in transit, so the connection is dropped.

> Older explanations often describe the client "encrypting the pre-master secret with the server's public key (RSA) and sending it." That is RSA key exchange, and it was removed in TLS 1.3. If the server's private key leaks later, all past traffic that was recorded can be decrypted. ECDHE creates a temporary key per connection and throws it away, so past traffic stays safe even if the private key leaks (forward secrecy).
{: .prompt-warning }

### 3-2. The TLS 1.3 Handshake

TLS 1.3 removes one round trip by having the client include its key exchange value in the first message.

```text
Client                                       Server
    |                                         |
    | --- ClientHello + key_share ----------> |  ECDHE public values sent up front
    |                                         |
    | <-- ServerHello + key_share ----------- |  server ECDHE public value
    |     (encrypted from here)               |
    | <-- EncryptedExtensions --------------- |
    | <-- Certificate ----------------------- |
    | <-- CertificateVerify ----------------- |  private key signature
    | <-- Finished -------------------------- |
    |                                         |   (end of 1 RTT)
    | --- Finished -------------------------> |
    | === HTTP request (encrypted) =========> |
```

Four things changed from TLS 1.2.

| Item | TLS 1.2 | TLS 1.3 |
| --- | --- | --- |
| Round trips to first request | 2 RTT | 1 RTT |
| RSA key exchange | Allowed | Removed |
| Certificate | Sent in plain text | Encrypted |
| Cipher suites | Combos incl. key exchange | Cipher + hash only, 5 |

If the round trip between server and client is 100 ms, TLS 1.3 sends the first HTTP request 100 ms earlier.

> TLS 1.3 also has a 0-RTT mode that sends data in the first message to a server you connected to before. But it cannot stop a replay attack, where an attacker records that message and sends it again. So only requests that give the same result when run more than once, such as GET, should go in 0-RTT.
{: .prompt-info }

### 3-3. Checking It Yourself

Connecting with `curl -v` prints the handshake along with the negotiated version and cipher suite.

```bash
curl -v https://example.com -o /dev/null
```

To see the certificate chain as well, use `openssl`.

```bash
openssl s_client -connect example.com:443 -servername example.com
```

## 4. Common Follow-up Questions

**Does switching to HTTPS make things slower?**

Each new connection costs extra handshake round trips: two for TLS 1.2, one for TLS 1.3. With keep-alive sending many requests over one connection, the handshake happens only once at the start. Data encryption runs on CPUs with AES hardware instructions (AES-NI), so most of the noticeable difference comes from the number of round trips.

**Is redirecting http to https enough?**

The first request goes out as plain HTTP, so intercepting that moment can block the redirect itself. Adding the `Strict-Transport-Security` header (HSTS) to responses makes the browser connect to that domain over HTTPS only, from the start, for the specified period.

```text
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

**Where is TLS terminated?**

A common setup has a web server such as Nginx decrypt TLS and pass plain HTTP to the application server behind it. The certificate only needs to be installed on the web server. This setup is covered in the [web server vs WAS post](/posts/webserver-vs-was-en/).

## Wrap-up

- HTTP sends requests and responses in plain text. HTTPS sends the same HTTP messages inside a TLS connection. The default ports are 80 and 443.
- TLS encrypts the content, detects altered data, and uses a certificate to check that the server owns the domain.
- Even over HTTPS, the IP and domain name you connect to are visible in between. The path, headers, cookies, and body are not.
- The handshake agrees on a secret with ECDHE, and the symmetric key derived from it encrypts all later data.
- TLS 1.2 takes 2 RTT from ClientHello to Finished. TLS 1.3 puts the key exchange value in ClientHello and takes 1 RTT.
- Every SSL version is prohibited. Even when people say "SSL handshake," it is actually a TLS 1.2 or 1.3 handshake.

## References

- [RFC 9110: HTTP Semantics (http and https URI schemes and default ports)](https://www.rfc-editor.org/rfc/rfc9110)
- [RFC 8446: The Transport Layer Security (TLS) Protocol Version 1.3](https://www.rfc-editor.org/rfc/rfc8446)
- [RFC 5246: The Transport Layer Security (TLS) Protocol Version 1.2](https://www.rfc-editor.org/rfc/rfc5246)
- [RFC 7568: Deprecating Secure Sockets Layer Version 3.0](https://www.rfc-editor.org/rfc/rfc7568)
- [RFC 8996: Deprecating TLS 1.0 and TLS 1.1](https://www.rfc-editor.org/rfc/rfc8996)
- [RFC 6797: HTTP Strict Transport Security (HSTS)](https://www.rfc-editor.org/rfc/rfc6797)
- [MDN: Transport Layer Security](https://developer.mozilla.org/en-US/docs/Web/Security/Transport_Layer_Security)
