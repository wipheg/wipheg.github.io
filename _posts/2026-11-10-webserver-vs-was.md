---
title: "웹 서버와 WAS의 차이, 그리고 Apache vs Nginx"
date: 2026-11-10 09:03:00 +0900
categories: [Backend, Infra]
tags: [web-server, was, apache, nginx, tomcat]
description: 웹 서버와 WAS가 각각 무슨 일을 하고 왜 나눠 쓰는지, Apache와 Nginx는 구조가 어떻게 다르고 어디에 쓰는지 정리했다.
toc: true
lang: ko-KR
ref: webserver-vs-was
---

백엔드 공부를 하다 보면 "웹 서버", "WAS", "Apache", "Nginx", "Tomcat" 같은 단어가 계속 나온다. 모두 HTTP 요청을 받는 서버라 비슷해 보이지만, 웹 서버는 이미 있는 파일을 돌려주고 WAS는 코드를 실행해서 응답을 새로 만든다.

> **요약**
> - 웹 서버는 정적 파일 제공과 요청 중계를, WAS는 애플리케이션 코드를 실행해 동적 응답을 만드는 일을 맡는다.
> - WAS 혼자서도 서비스는 돌아가지만 성능, 보안, 확장성, 장애 격리 때문에 웹 서버를 앞에 둔다.
> - Apache는 연결마다 프로세스나 스레드를 붙이는 구조다. 디렉터리별 설정(`.htaccess`)과 모듈로 기능을 붙이기 쉽다.
> - Nginx는 이벤트 기반 구조라 적은 자원으로 많은 동시 접속을 처리한다. 리버스 프록시와 로드 밸런서로 많이 쓴다.
{: .prompt-tip }

## 1. 정적 콘텐츠와 동적 콘텐츠

웹 서버와 WAS는 처리하는 콘텐츠가 정적이냐 동적이냐로 나뉜다.

| 구분 | 정적 콘텐츠 | 동적 콘텐츠 |
| --- | --- | --- |
| 정의 | 미리 만들어져 있는 파일 | 요청할 때마다 로직을 실행해서 만드는 결과 |
| 예시 | HTML, CSS, JS, 이미지, 폰트 | 로그인한 사용자 정보, 검색 결과, 주문 내역 |
| 특징 | 누가 요청해도 응답이 같음 | 요청 내용, 사용자, DB 상태에 따라 응답이 바뀜 |

정적 콘텐츠는 웹 서버가 디스크에서 파일을 읽어 그대로 보내고, 동적 콘텐츠는 WAS가 코드를 실행해서 만든다.

## 2. 웹 서버

웹 서버는 HTTP 요청을 받아 정적 파일을 그대로 돌려주는 서버다. 동적인 처리가 필요한 요청은 직접 처리하지 않고 뒤쪽의 WAS로 넘긴다.

웹 서버가 맡는 일은 크게 다섯 가지다.

- 정적 파일 제공 (HTML, CSS, JS, 이미지)
- 리버스 프록시: 클라이언트 요청을 뒤쪽 서버(WAS)로 전달
- 로드 밸런싱: 여러 WAS에 요청 분산
- SSL/TLS 종료: HTTPS 암호화와 복호화를 대신 처리
- 캐싱, gzip 압축, 접근 제어, 요청 제한(Rate Limit)

대표 제품으로는 Apache HTTP Server, Nginx, Caddy, LiteSpeed, IIS가 있다.

## 3. WAS

WAS(Web Application Server)는 DB 조회나 비즈니스 로직 같은 애플리케이션 코드를 실행해서 동적 콘텐츠를 만드는 서버다. Java에서는 서블릿과 JSP를 실행하는 서블릿 컨테이너가 여기에 해당한다.

WAS가 맡는 일은 크게 네 가지다.

- 비즈니스 로직 실행
- DB 연동과 트랜잭션 처리
- 세션 관리
- 스레드 풀과 커넥션 풀 관리

대표 제품은 Java 쪽의 Tomcat, Jetty, Undertow, WildFly(JBoss), WebLogic, JEUS다. Python의 Gunicorn이나 uWSGI, PHP의 PHP-FPM도 비슷한 역할을 한다.

> Tomcat에는 HTTP 요청을 받는 커넥터(Coyote)가 들어 있어서 브라우저 요청을 직접 받을 수 있고 정적 파일도 줄 수 있다. 그러면 Nginx 없이 Tomcat만 띄워도 되지 않을까? 다음 절이 그 답이다.
{: .prompt-info }

## 4. 그런데 왜 나눠서 쓸까

WAS 혼자서도 서비스는 돌아간다. 그래도 실무에서는 대부분 웹 서버를 WAS 앞에 둔다.

```text
Client ──▶ Web Server (Nginx) ──▶ WAS (Tomcat) ──▶ DB
              │
              └─ 정적 파일은 여기서 바로 응답
```

**부하를 나눈다.** Tomcat은 요청 하나를 스레드 하나로 처리하고, 쓸 수 있는 스레드 수에 상한이 있다(Spring Boot 내장 Tomcat 기본값 200개). 이미지나 CSS 요청까지 Tomcat이 받으면 그만큼 스레드가 파일 전송에 묶이고, 정작 로직이 필요한 요청이 스레드를 기다리게 된다. 정적 파일은 웹 서버가 보내고 Tomcat 스레드는 로직이 필요한 요청에만 쓴다.

**보안.** WAS를 외부에 직접 노출하지 않고 웹 서버 뒤에 숨길 수 있다. 외부에 열리는 포트는 웹 서버뿐이고 WAS는 내부망에만 둔다.

**로드 밸런싱과 무중단 배포.** 웹 서버 하나가 WAS 여러 대에 요청을 나눠 준다. 배포할 때 WAS를 한 대씩 빼고 넣으면 서비스를 멈추지 않고 배포할 수 있다.

**장애 격리.** WAS가 죽어도 웹 서버는 살아 있으므로 점검 페이지나 에러 페이지를 보여줄 수 있다.

**SSL, 압축, 캐싱을 한 곳에서 처리.** HTTPS 인증서는 웹 서버에만 설치하고, 웹 서버와 WAS 사이는 내부망에서 HTTP로 통신한다. WAS가 여러 대여도 인증서를 갱신할 곳은 웹 서버 하나뿐이다.

> Spring Boot의 내장 Tomcat을 쓰면 Nginx가 필요 없을까? 인스턴스 한 대로 HTTP만 쓰는 서비스라면 없어도 된다. 하지만 HTTPS, 여러 인스턴스 로드 밸런싱, 무중단 배포, 정적 파일 캐싱 같은 요구가 생기면 앞단에 Nginx나 클라우드 로드 밸런서를 두는 게 일반적이다.
{: .prompt-tip }

## 5. 웹 서버와 WAS 비교

| 항목 | 웹 서버 | WAS |
| --- | --- | --- |
| 주 목적 | 정적 콘텐츠 제공, 요청 중계 | 동적 콘텐츠 생성 |
| 처리 대상 | HTML, CSS, JS, 이미지 | 비즈니스 로직, DB 연동 |
| 대표 제품 | Nginx, Apache HTTP Server | Tomcat, Jetty, Undertow |
| 위치 | 클라이언트와 가까운 앞단 | 웹 서버 뒤, 내부망 |
| 실행 환경 | C로 작성된 네이티브 프로그램 | JVM, Python 같은 언어 런타임 위에서 실행 |

## 6. Apache와 Nginx

웹 서버로 가장 많이 쓰는 두 가지가 Apache HTTP Server와 Nginx다.

> "Apache"는 보통 Apache HTTP Server(httpd)를 말한다. Tomcat도 Apache 재단 프로젝트라 "Apache Tomcat"이라고 부르지만 둘은 완전히 다른 소프트웨어다.
{: .prompt-warning }

### 6-1. 배경

Apache는 1995년에 나와 오랫동안 웹 서버 시장 1위를 지켰다. URL 재작성, SSL, 인증 같은 기능을 모듈로 붙여 쓴다.

Nginx는 2004년에 공개됐다. 서버 한 대로 동시 접속 1만 개를 처리하는 문제, 이른바 C10K 문제를 풀려고 만들었다. W3Techs 같은 웹 서버 점유율 통계에서 지금은 Nginx가 Apache보다 높다.

### 6-2. 요청 처리 구조

동시 접속 처리량과 메모리 사용량의 차이가 여기서 나온다.

**Apache는 프로세스/스레드 기반이다.** 연결 하나를 프로세스나 스레드 하나가 맡는다. 구조는 MPM(Multi-Processing Module)으로 고른다.

| MPM | 방식 | 특징 |
| --- | --- | --- |
| prefork | 연결 1개 = 프로세스 1개 | 안정적, 메모리 많이 씀, mod_php와 호환 |
| worker | 프로세스 1개에 스레드 여러 개 | prefork보다 메모리 효율이 좋음 |
| event | worker + Keep-Alive 연결 분리 처리 | 유휴 연결 부담이 적음, 2.4 권장 |

```text
요청 A ──▶ [스레드 1]
요청 B ──▶ [스레드 2]
요청 C ──▶ [스레드 3]   ← 접속이 늘수록 스레드와 메모리도 같이 는다
```

요청 하나가 프로세스나 스레드 하나에 묶여 있어 처리 흐름을 따라가기 쉽다. prefork라면 프로세스 하나가 죽어도 다른 프로세스가 맡은 요청은 영향을 받지 않는다. 대신 동시 접속이 1만 개면 프로세스나 스레드도 그만큼 필요해서 메모리 사용량과 컨텍스트 스위칭 비용이 커진다.

**Nginx는 이벤트 기반, 비동기 논블로킹이다.** 마스터 프로세스 1개와 워커 프로세스 몇 개(보통 CPU 코어 수만큼)로 동작한다. 워커 하나가 이벤트 루프를 돌면서 수천~수만 개의 연결을 동시에 처리한다.

```text
             ┌─ 요청 A
[워커 1] ────┼─ 요청 B    ← 준비된 이벤트부터 처리, 기다리는 동안 다른 요청 처리
             └─ 요청 C ...
```

응답을 기다리는 연결에 스레드를 붙여 두지 않으므로 연결이 수만 개 열려 있어도 메모리가 크게 늘지 않는다. 그래서 정적 파일 전송과 프록시처럼 기다리는 시간이 긴 일에 강하다. 대신 워커 안에서 오래 걸리는 블로킹 작업을 하면 그 워커가 맡은 연결이 모두 함께 멈춘다.

### 6-3. 동적 콘텐츠 처리

Apache는 `mod_php` 같은 모듈로 서버 안에서 직접 PHP를 실행할 수 있다. PHP 파일을 올려 두기만 하면 Apache가 바로 실행하므로 별도 프로세스를 띄울 필요가 없다.

Nginx는 동적 콘텐츠를 직접 처리하지 않고 외부 프로세스에 넘긴다.

- PHP → PHP-FPM (FastCGI)
- Java → Tomcat (`proxy_pass`)
- Python → Gunicorn, uWSGI

PHP-FPM 같은 프로세스를 따로 띄우고 연결 설정을 해야 하는 번거로움이 있다. 대신 웹 서버와 애플리케이션을 따로 재시작하고 따로 늘릴 수 있다.

### 6-4. 설정 방식

| 항목 | Apache | Nginx |
| --- | --- | --- |
| 디렉터리별 설정 | `.htaccess`로 덮어쓰기 가능 | 없음, 메인 설정에서 일괄 관리 |
| 장점 | 관리자 권한 없이 설정 변경 | 파일 탐색 없음, 설정이 한 곳에 모임 |
| 단점 | 요청마다 디렉터리 탐색, 성능 손해 | 설정 변경에 서버 권한 필요 |

### 6-5. 모듈

Apache는 실행 중에 모듈을 동적으로 불러올 수 있고 모듈 종류가 아주 많다(`mod_rewrite`, `mod_ssl`, `mod_security` 등).

Nginx는 원래 컴파일할 때 모듈을 넣어야 했다. 1.9.11부터 동적 모듈을 지원하지만 생태계는 Apache보다 작다.

## 7. Apache와 Nginx 비교

| 항목 | Apache | Nginx |
| --- | --- | --- |
| 구조 | 프로세스/스레드 기반 (MPM) | 이벤트 기반, 비동기 논블로킹 |
| 동시 접속 처리 | 접속 수에 비례해 자원 증가 | 적은 자원으로 대량 처리 |
| 유휴 연결 | 스레드를 점유 (event MPM은 완화) | 이벤트 루프가 처리, 부담 적음 |
| 동적 콘텐츠 | 모듈로 직접 처리 가능 | 외부 프로세스로 위임 |
| 설정 | 디렉터리별 `.htaccess` 가능 | 메인 설정 파일에서만 관리 |
| 모듈 | 실행 중 로드, 종류 많음 | 1.9.11부터 동적 모듈, 종류 적음 |
| 주 사용처 | 레거시, 공유 호스팅, PHP | 리버스 프록시, 로드 밸런서, 고트래픽 |

> Apache도 event MPM에서는 Keep-Alive로 열려 있기만 한 연결을 별도 스레드가 모아서 맡는다. 그래서 유휴 연결 때문에 작업 스레드가 묶이는 문제가 많이 줄었다. "Nginx가 무조건 빠르다"보다는 "많은 동시 접속과 프록시는 Nginx, 디렉터리별 설정과 모듈 확장은 Apache가 편하다"고 이해하는 게 정확하다.
{: .prompt-info }

## 8. 언제 무엇을 쓸까

**Nginx를 고르는 경우**

- 트래픽과 동시 접속이 많은 서비스
- Spring Boot, Node.js, Django 같은 애플리케이션 앞단의 리버스 프록시
- 로드 밸런싱, SSL 종료, 정적 파일 서빙
- 컨테이너나 클라우드 환경 (설정 파일 하나로 프록시까지 끝난다)

**Apache를 고르는 경우**

- `.htaccess`가 필요한 공유 호스팅 환경
- WordPress 같은 PHP 기반의 전통적인 LAMP 스택
- 이미 Apache로 운영 중인 레거시 시스템
- 특정 Apache 모듈이 꼭 필요한 경우

**둘을 같이 쓰는 경우**

앞단의 Nginx가 정적 파일과 프록시를 맡고 뒷단의 Apache가 동적 처리를 하는 구성도 있다. 기존 Apache 설정과 `.htaccess`는 그대로 두고 정적 파일과 SSL 처리만 Nginx로 넘기고 싶을 때 쓴다.

```text
Client ──▶ Nginx (정적 파일, SSL, 프록시) ──▶ Apache + PHP
```

## 9. 예시: Nginx + Spring Boot 리버스 프록시

Spring Boot 인스턴스 두 대(8080, 8081 포트) 앞에 Nginx를 두는 최소 설정이다.

```nginx
upstream app_servers {
    server 127.0.0.1:8080;
    server 127.0.0.1:8081;   # 인스턴스 2대로 로드 밸런싱
}

server {
    listen 80;
    server_name example.com;

    # 정적 파일은 Nginx가 직접 응답
    location /static/ {
        root /var/www;
        expires 7d;
    }

    # 나머지 요청은 WAS로 전달
    location / {
        proxy_pass http://app_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

> Nginx를 거치면 WAS가 보는 접속 IP는 항상 Nginx의 IP(127.0.0.1)다. 실제 클라이언트 IP는 위처럼 `X-Forwarded-For` 헤더로 넘겨야 하고, Spring Boot는 `server.forward-headers-strategy`를 설정해야 이 헤더를 읽는다.
{: .prompt-warning }

## 정리

- 웹 서버는 파일을 그대로 돌려주고 요청을 뒤로 넘긴다. WAS는 코드를 실행해 응답을 만든다.
- WAS 혼자서도 동작하지만 WAS 스레드 절약, WAS 비노출, 여러 대 분산, 장애 시 에러 페이지 때문에 웹 서버를 앞에 둔다.
- Apache는 연결마다 프로세스나 스레드를 쓴다. `.htaccess`가 필요한 공유 호스팅과 기존 PHP 환경에서 많이 쓴다.
- Nginx는 워커 몇 개가 이벤트 루프로 연결 수만 개를 처리한다. 리버스 프록시와 로드 밸런서로 많이 쓴다.
- 새로 구성하는 서비스라면 보통 Nginx 뒤에 WAS를 두는 구성에서 시작한다.

## 참고 문서

- [Apache HTTP Server 2.4: Multi-Processing Modules (MPMs)](https://httpd.apache.org/docs/2.4/mpm.html)
- [Apache HTTP Server 2.4: .htaccess files](https://httpd.apache.org/docs/2.4/howto/htaccess.html)
- [nginx documentation](https://nginx.org/en/docs/)
- [nginx: Beginner's Guide](https://nginx.org/en/docs/beginners_guide.html)
- [The C10K problem](http://www.kegel.com/c10k.html)
