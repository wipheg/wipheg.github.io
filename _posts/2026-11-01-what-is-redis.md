---
title: "Redis란 무엇인가"
date: 2026-11-01 09:00:00 +0900
categories: [Database, Redis]
tags: [redis, cache, in-memory, nosql]
description: Redis가 데이터를 어디에 어떻게 저장하는지, 어떤 자료구조를 제공하는지, 메모리가 차거나 서버가 꺼지면 어떻게 되는지, 실무에서 어디에 쓰는지 정리했다.
toc: true
lang: ko-KR
ref: what-is-redis
---

백엔드 구성도를 보면 DB 옆에 Redis가 붙어 있는 경우가 많다. "캐시로 쓴다", "세션을 넣는다"는 말은 자주 듣지만 Redis가 데이터를 어디에 두는지, 서버가 꺼지면 데이터가 남는지는 따로 알아봐야 한다. 이 글은 Redis가 데이터를 저장하고 지우고 디스크에 남기는 방식을 순서대로 정리한다.

> **요약**
> - Redis는 모든 데이터를 메모리(RAM)에 두는 키-값 저장소다. 읽기와 쓰기에서 디스크를 거치지 않는다.
> - 값으로 문자열뿐 아니라 List, Hash, Set, Sorted Set 같은 자료구조를 저장하고 자료구조마다 전용 명령이 있다.
> - 명령은 주로 스레드 하나가 차례로 실행한다. 그래서 오래 걸리는 명령 하나(`KEYS *` 등)가 그동안 들어온 다른 요청을 모두 기다리게 한다.
> - 기본 설정에서 메모리가 가득 차면 키를 지우지 않고 쓰기 명령에 에러를 돌려준다(`noeviction`). 캐시로 쓸 때는 `maxmemory-policy`를 바꿔야 한다.
> - 디스크 저장은 RDB(주기적 스냅샷)와 AOF(쓰기 명령 기록) 두 가지다. 기본 설정에서 AOF는 꺼져 있다.
{: .prompt-tip }

## 먼저 알아둘 단어

| 단어 | 뜻 |
| --- | --- |
| 인메모리 | 데이터를 디스크가 아니라 RAM에 두고 읽고 쓰는 방식 |
| 키-값 저장소 | 키 하나로 값 하나를 찾는 저장소. SQL과 테이블이 없다 |
| TTL | 키가 자동으로 지워질 때까지 남은 시간 |
| 영속성 | 서버가 꺼졌다 켜져도 데이터가 남는 성질 |
| 축출(eviction) | 메모리가 모자랄 때 기존 키를 골라 지우는 동작 |

## 1. Redis는 무엇인가

Redis는 데이터를 메모리에 저장하는 키-값 저장소다. 클라이언트는 TCP로 접속해서(기본 포트 6379) `SET`, `GET` 같은 명령을 보낸다.

```text
127.0.0.1:6379> SET user:42:name "kim"
OK
127.0.0.1:6379> GET user:42:name
"kim"
```

MySQL 같은 관계형 DB와는 데이터를 찾는 방법이 다르다.

| 항목 | MySQL (InnoDB) | Redis |
| --- | --- | --- |
| 데이터 위치 | 디스크, 자주 쓰는 페이지만 메모리에 캐시 | 전부 메모리 |
| 조회 방법 | SQL, 인덱스, 조인 | 키로만 조회 |
| 데이터 크기 한계 | 디스크 용량 | 메모리 용량 |
| 서버 재시작 | 데이터 유지 | 설정한 디스크 저장 방식에 따라 다름 |

Redis에는 "나이가 30 이상인 사용자" 같은 조건 검색이 없다. 값을 꺼내려면 키를 알아야 한다. 그래서 키 이름을 `user:42:name`처럼 `종류:ID:필드` 형태로 정해 두고 코드에서 키를 조립해 쓰는 경우가 많다.

## 2. 명령은 스레드 하나가 차례로 실행한다

Redis는 클라이언트가 수천 개 붙어 있어도 명령 실행은 주로 메인 스레드 하나가 맡는다. 명령이 하나씩 차례로 실행되므로 두 클라이언트가 같은 키를 동시에 `INCR`해도 값이 꼬이지 않는다. 락을 걸 필요가 없다.

```text
클라이언트 A ── INCR counter ─┐
클라이언트 B ── INCR counter ─┼──▶ [메인 스레드] 한 번에 하나씩 실행
클라이언트 C ── GET counter  ─┘        counter: 1 → 2 → 읽기
```

대신 명령 하나가 오래 걸리면 그동안 다른 명령은 전부 기다린다. 가장 흔한 예가 `KEYS`다. `KEYS`는 키 개수(N)만큼 시간이 드는 O(N) 명령이라 키가 수백만 개인 서버에서 `KEYS *`를 실행하면 그동안 다른 요청이 모두 멈춘다.

> 공식 문서는 `KEYS`를 일반 애플리케이션 코드에서 쓰지 말라고 안내한다. 키 목록이 필요하면 커서로 조금씩 나눠 가져오는 `SCAN`을 쓴다. 원소가 많은 Hash, Set을 한 번에 다 가져오는 `HGETALL`, `SMEMBERS`도 같은 이유로 주의한다.
{: .prompt-warning }

> Redis 6.0부터는 클라이언트 소켓 읽기와 쓰기, 요청 파싱을 별도 I/O 스레드에 맡기는 `io-threads` 설정이 있다. 기본값은 꺼져 있어서 메인 스레드만 쓴다.
{: .prompt-info }

## 3. 자료구조

Redis의 값은 문자열 하나에 그치지 않는다. 자료구조마다 전용 명령이 있어서 애플리케이션이 값을 꺼내 고친 뒤 다시 넣지 않고 Redis 안에서 바로 바꾼다.

| 타입 | 저장 형태 | 주요 명령 | 쓰는 곳 |
| --- | --- | --- | --- |
| String | 문자열, 숫자 | `SET` `GET` `INCR` | 캐시, 카운터 |
| List | 순서 있는 목록 | `LPUSH` `RPOP` `LRANGE` | 작업 큐, 최근 목록 |
| Hash | 필드-값 묶음 | `HSET` `HGET` | 객체 저장 |
| Set | 중복 없는 집합 | `SADD` `SISMEMBER` | 중복 체크, 태그 |
| Sorted Set | 점수 순 정렬 집합 | `ZADD` `ZRANGE` | 랭킹 |

String 값 하나는 최대 512MB까지 저장한다. 이 밖에 Stream(로그형 메시지), Bitmap, HyperLogLog(근사 개수 세기), Geo(위치) 타입도 있다.

예를 들어 게임 점수 랭킹은 Sorted Set 하나로 만든다. 점수를 넣으면 Redis가 정렬된 상태를 유지하므로 상위 3명을 볼 때 전체를 다시 정렬하지 않는다.

```text
127.0.0.1:6379> ZADD ranking 1500 "kim" 2300 "lee" 1800 "park"
(integer) 3
127.0.0.1:6379> ZREVRANGE ranking 0 2 WITHSCORES
1) "lee"
2) "2300"
3) "park"
4) "1800"
5) "kim"
6) "1500"
```

## 4. 만료 시간(TTL)

키에 만료 시간을 걸면 그 시간이 지난 뒤 Redis가 키를 지운다. 캐시와 세션처럼 일정 시간만 유효한 데이터에 쓴다.

```text
127.0.0.1:6379> SET session:abc123 "user:42" EX 1800   # 30분 뒤 만료
OK
127.0.0.1:6379> TTL session:abc123
(integer) 1799
```

만료된 키를 지우는 방식은 두 가지다.

- **접근할 때 지운다(passive).** 클라이언트가 키를 읽으려 할 때 만료됐으면 그때 지우고 없는 키로 응답한다.
- **주기적으로 표본을 골라 지운다(active).** 만료 시간이 있는 키 중 일부를 무작위로 골라 검사하고 만료된 키를 지운다.

그래서 만료 시간이 지난 키가 아무도 읽지 않는 동안 잠깐 메모리에 남아 있을 수 있다. 응답에는 나타나지 않는다.

## 5. 메모리가 가득 차면

Redis는 데이터를 메모리에만 두므로 쓸 수 있는 메모리 양을 `maxmemory`로 정한다. 이 한도에 닿았을 때의 동작은 `maxmemory-policy`가 결정한다.

| 정책 | 지우는 대상 |
| --- | --- |
| `noeviction` (기본값) | 지우지 않음. 쓰기 명령에 에러 반환 |
| `allkeys-lru` | 전체 키 중 가장 오래 안 쓴 키 |
| `allkeys-lfu` | 전체 키 중 가장 적게 쓴 키 |
| `volatile-lru` | TTL 있는 키 중 가장 오래 안 쓴 키 |
| `volatile-ttl` | TTL 있는 키 중 만료가 가장 가까운 키 |
| `allkeys-random` | 전체 키 중 무작위 |

이 밖에 `volatile-lfu`, `volatile-random`이 있고, Redis 8.6부터 `allkeys-lrm`, `volatile-lrm`이 추가됐다.

> 기본값 `noeviction`에서는 메모리가 차면 `SET` 같은 쓰기 명령이 `-OOM`으로 시작하는 에러를 돌려준다. 캐시 용도라면 `allkeys-lru`나 `allkeys-lfu`로 바꿔야 오래된 캐시가 지워지고 새 캐시가 들어간다. 반대로 세션처럼 지워지면 안 되는 데이터와 캐시를 한 Redis에 섞어 두면, 축출 정책이 세션까지 지울 수 있다.
{: .prompt-warning }

## 6. 디스크에 저장하기 (영속성)

메모리 데이터는 프로세스가 꺼지면 사라진다. Redis는 데이터를 디스크에 남기는 방법 두 가지를 제공한다.

### 6-1. RDB: 주기적 스냅샷

RDB는 정해 둔 간격마다 그 순간의 데이터 전체를 파일 하나(`dump.rdb`)로 저장한다. 기본 저장 규칙은 다음 세 조건 중 하나를 만족할 때 저장하는 것이다.

- 3600초(1시간) 안에 키가 1번 이상 바뀜
- 300초(5분) 안에 키가 100번 이상 바뀜
- 60초 안에 키가 10000번 이상 바뀜

파일 하나라 백업과 복원이 쉽고 재시작할 때 AOF보다 빨리 읽어 들인다. 대신 마지막 스냅샷 이후의 변경은 장애가 나면 잃는다. 위 규칙이라면 변경이 적은 서버에서 최대 1시간치를 잃을 수 있다.

### 6-2. AOF: 쓰기 명령 기록

AOF(Append Only File)는 데이터를 바꾸는 명령을 받을 때마다 파일 끝에 덧붙인다. 재시작할 때 이 명령을 처음부터 다시 실행해서 데이터를 복원한다.

AOF를 디스크에 실제로 내려쓰는(fsync) 주기는 `appendfsync`로 정한다. 기본값이자 공식 문서 권장값은 `everysec`(1초마다)이고 이때 장애가 나면 최대 1초치 쓰기를 잃는다.

```conf
# redis.conf
appendonly yes          # 기본값은 no
appendfsync everysec    # 기본값
```

### 6-3. 비교

| 항목 | RDB | AOF |
| --- | --- | --- |
| 저장 내용 | 특정 시점의 데이터 전체 | 쓰기 명령 기록 |
| 잃을 수 있는 양 | 마지막 스냅샷 이후 전부 | `everysec` 기준 약 1초 |
| 파일 크기 | AOF보다 작음 | 명령이 쌓일수록 커짐, 주기적으로 재작성 |
| 기본 설정 | 켜짐 | 꺼짐 (`appendonly no`) |

데이터를 잃어도 DB에서 다시 채우면 되는 캐시라면 RDB만 쓰거나 둘 다 끄기도 한다. 세션이나 큐처럼 Redis에만 있는 데이터라면 AOF를 켠다. 공식 문서는 PostgreSQL 수준의 안전성이 필요하면 둘을 함께 쓰라고 안내한다.

## 7. 어디에 쓰는가

### 7-1. 캐시

가장 많이 쓰는 용도다. DB 조회 결과를 Redis에 넣어 두고 같은 요청이 오면 DB 대신 Redis에서 돌려준다. 애플리케이션이 캐시를 먼저 보고 없으면 DB를 읽어 채우는 방식을 캐시 어사이드(cache-aside)라고 한다.

```text
요청 ──▶ App ──① GET product:7 ──▶ Redis
          │                         │
          │◀── ② 있으면 바로 응답 ───┘
          │
          ├── ③ 없으면 DB 조회 ──▶ MySQL
          │
          └── ④ SET product:7 ... EX 600 ──▶ Redis
```

캐시에는 TTL을 꼭 건다. TTL이 없으면 DB 값이 바뀐 뒤에도 옛 값이 계속 응답되고 쓰지 않는 키가 메모리에 계속 쌓인다.

### 7-2. 세션 저장소

서버가 여러 대일 때 세션을 각 서버 메모리에 두면, 로그인한 서버와 다른 서버로 요청이 가는 순간 로그인이 풀린다. 세션을 Redis에 두면 모든 서버가 같은 세션을 읽는다.

### 7-3. 요청 횟수 제한

`INCR`로 사용자별 요청 수를 세고 첫 요청에서 만료 시간을 건다. 아래는 1분에 100번까지 허용하는 예다.

```text
INCR   rate:user:42      # 결과가 1이면 이번이 첫 요청
EXPIRE rate:user:42 60   # 첫 요청일 때만 60초 만료 설정
                         # INCR 결과가 100을 넘으면 요청 거절
```

### 7-4. 랭킹

3절의 Sorted Set 예시처럼 점수를 넣으면 정렬 상태가 유지된다. `ZREVRANK`로 특정 사용자의 순위도 바로 구한다.

### 7-5. 분산 락

서버 여러 대가 같은 작업을 동시에 하지 않도록, 키가 없을 때만 저장하는 `SET ... NX`로 락을 잡는다.

```text
SET lock:order:1001 "a7f3-uuid" NX PX 30000   # 없을 때만 저장, 30초 뒤 자동 해제
```

> 락을 풀 때 `DEL`만 하면 안 된다. 내 락이 30초가 지나 풀리고 다른 서버가 새로 잡은 뒤라면, 남의 락을 지우게 된다. 값(`a7f3-uuid`)이 내가 넣은 값인지 확인하고 지우는 과정을 Lua 스크립트로 묶어 한 번에 실행한다.
{: .prompt-warning }

## 8. 여러 대로 늘리기

| 구성 | 하는 일 |
| --- | --- |
| 복제(Replication) | 마스터의 데이터를 레플리카가 복사. 읽기 분산, 백업 |
| Sentinel | 마스터 장애를 감지해 레플리카를 새 마스터로 승격 |
| Cluster | 데이터를 여러 마스터에 나눠 저장 |

Redis Cluster는 키 공간을 16384개 해시 슬롯으로 나누고 슬롯을 마스터들에게 나눠 준다. 키가 어느 슬롯에 들어갈지는 `CRC16(key) mod 16384`로 정한다.

> 클러스터에서는 `MSET`처럼 키 여러 개를 다루는 명령이 모든 키가 같은 슬롯에 있을 때만 동작한다. 키 이름에 `{}`(해시 태그)를 넣으면 괄호 안의 문자열로만 슬롯을 계산한다. `{user1000}.following`과 `{user1000}.followers`는 같은 슬롯에 들어간다.
{: .prompt-warning }

## 9. 예시: Spring Boot에서 쓰기

`spring-boot-starter-data-redis`를 추가하고 접속 정보를 적는다.

```yaml
# application.yml
spring:
  data:
    redis:
      host: localhost
      port: 6379
```

```java
@Service
@RequiredArgsConstructor
public class ProductService {

    private final StringRedisTemplate redis;
    private final ProductRepository repository;

    public String getProductName(long id) {
        String key = "product:" + id + ":name";

        String cached = redis.opsForValue().get(key);
        if (cached != null) {
            return cached;                      // 캐시에 있으면 바로 반환
        }

        String name = repository.findById(id)
                .orElseThrow()
                .getName();                     // 없으면 DB 조회
        redis.opsForValue().set(key, name, Duration.ofMinutes(10)); // TTL 10분
        return name;
    }
}
```

> Spring Boot 3부터 설정 키가 `spring.redis.*`에서 `spring.data.redis.*`로 바뀌었다. 블로그 예제의 `spring.redis.host`를 Spring Boot 3에 그대로 쓰면 무시되고 기본값 `localhost:6379`로 접속한다.
{: .prompt-warning }

## 정리

- Redis는 데이터를 모두 메모리에 두는 키-값 저장소다. 키로만 조회하고 데이터 크기는 메모리 용량을 넘지 못한다.
- 명령은 메인 스레드 하나가 차례로 실행한다. 락 없이 `INCR`이 안전하지만 `KEYS *` 같은 O(N) 명령 하나가 그동안 모든 요청을 멈추게 한다.
- String, List, Hash, Set, Sorted Set마다 전용 명령이 있어서 카운터, 큐, 랭킹을 Redis 안에서 바로 처리한다.
- 기본 설정(`noeviction`)은 메모리가 차면 쓰기를 거부한다. 캐시로 쓰면 `allkeys-lru`나 `allkeys-lfu`로 바꾼다.
- 기본 설정은 RDB 스냅샷만 켜져 있어서 장애 시 마지막 스냅샷 이후 데이터를 잃는다. Redis에만 있는 데이터라면 AOF(`everysec`, 최대 약 1초 손실)를 켠다.

Redis를 도입할 때 확인할 것:

1. 이 Redis에 들어갈 데이터가 지워져도 되는 캐시인가, Redis에만 있는 데이터인가
2. `maxmemory`와 `maxmemory-policy`를 용도에 맞게 정했는가
3. 캐시 키에 TTL을 걸었는가
4. 코드에 `KEYS`가 없는가
5. 영속성(RDB, AOF) 설정이 용도에 맞는가

## 참고 문서

- [Redis: Data types](https://redis.io/docs/latest/develop/data-types/)
- [Redis: Strings](https://redis.io/docs/latest/develop/data-types/strings/)
- [Redis: KEYS](https://redis.io/docs/latest/commands/keys/)
- [Redis: EXPIRE](https://redis.io/docs/latest/commands/expire/)
- [Redis: Key eviction](https://redis.io/docs/latest/develop/reference/eviction/)
- [Redis: Persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)
- [Redis Cluster specification](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/)
- [redis.conf (기본 설정 파일)](https://raw.githubusercontent.com/redis/redis/unstable/redis.conf)
- [Spring Boot 3.0 Migration Guide: Redis Properties](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide)
- [Spring Boot: Working with NoSQL Technologies (Redis)](https://docs.spring.io/spring-boot/reference/data/nosql.html)
