---
title: "UUID를 PK로 쓰면 왜 INSERT가 느려질까?"
date: 2026-11-10 06:30:00 +0900
categories: [Database, MySQL]
tags: [mysql, innodb, uuid, primary-key, index]
description: 기본 키를 UUID로 바꾸면 INSERT가 느려지고 테이블이 커진다는 말이 있다. MySQL 8.4 공식 문서와 UUID 표준을 따라가며 이유를 정리했다.
toc: true
lang: ko-KR
ref: uuid-pk-insert
---

```sql
CREATE TABLE orders (
  id CHAR(36) PRIMARY KEY,   -- '6ccd780c-baba-1026-9564-5b8c656024db' 같은 값
  ...
);
```

UUID는 여러 서버가 각자 만들어도 겹치지 않도록 설계된 128비트 식별자다. 주문 번호를 1, 2, 3 대신 UUID로 쓰면 데이터베이스에 번호를 받으러 가지 않고도 애플리케이션이 번호를 먼저 정할 수 있다. 그런데 InnoDB 테이블의 기본 키(PK)를 UUID로 바꾸면 INSERT가 느려지고 테이블이 커진다는 이야기를 자주 듣는다.

> **요약**
> - InnoDB는 행을 PK 순서대로 정리해서 저장한다. PK가 1, 2, 3처럼 커지기만 하면 새 행은 늘 맨 끝에 붙는다.
> - UUIDv4처럼 순서가 없는 값을 넣으면 새 행이 매번 중간 어딘가로 들어간다. 꽉 찬 페이지는 둘로 나뉘고 페이지는 1/2에서 15/16만 찬다. 번호순으로 넣으면 15/16쯤 찬다.
> - 보조 인덱스에는 PK 값이 함께 들어간다. PK가 8바이트(`BIGINT`)에서 36바이트(`CHAR(36)`)로 커지면 모든 보조 인덱스가 같이 커진다.
> - UUID를 꼭 써야 하면 `BINARY(16)`에 저장하고 시간 순서로 정렬되는 값을 쓴다. MySQL의 `UUID()`라면 `UUID_TO_BIN(uuid, 1)`, 애플리케이션에서 만든다면 UUIDv7이다.
{: .prompt-tip }

## 먼저 알아둘 단어

| 단어 | 한 줄 설명 |
| --- | --- |
| 행 | 테이블에 들어 있는 데이터 한 줄. 여기서는 주문 하나 |
| 기본 키 (PK) | 행마다 하나씩 붙는, 겹치지 않는 값 |
| 페이지 | InnoDB가 디스크와 메모리 사이에서 한 번에 옮기는 단위. 기본 16KB |
| 클러스터드 인덱스 | 행 전체를 PK 순서로 정리해 둔 구조. 테이블 데이터 그 자체 |
| 보조 인덱스 | PK가 아닌 다른 컬럼 값으로 정리한 목록. 행을 찾아가려고 PK를 같이 적어 둔다 |
| 페이지 분할 | 꽉 찬 페이지에 행을 넣으려고 페이지를 둘로 나누는 일 |
| 버퍼 풀 | InnoDB가 디스크에서 읽은 페이지를 올려 두는 메모리 |

## 예시 테이블

설명에는 아래 두 테이블을 쓴다. 컬럼 구성은 같고 PK만 다르다.

```sql
-- 1) 번호가 1, 2, 3 순서로 커지는 PK
CREATE TABLE orders_seq (
  id         BIGINT AUTO_INCREMENT PRIMARY KEY,
  user_id    INT NOT NULL,
  amount     INT NOT NULL,
  created_at DATETIME NOT NULL,
  KEY idx_user_id (user_id),
  KEY idx_created_at (created_at)
) ENGINE=InnoDB;

-- 2) UUID 문자열을 그대로 넣는 PK
CREATE TABLE orders_uuid (
  id         CHAR(36) PRIMARY KEY,   -- 애플리케이션이 만든 UUIDv4 값
  user_id    INT NOT NULL,
  amount     INT NOT NULL,
  created_at DATETIME NOT NULL,
  KEY idx_user_id (user_id),
  KEY idx_created_at (created_at)
) ENGINE=InnoDB;
```

## 1. InnoDB는 PK 순서로 행을 저장한다

InnoDB 테이블에는 클러스터드 인덱스라는 특별한 인덱스가 하나씩 있다. 공식 문서는 이 인덱스에 행 데이터가 들어 있고 보통은 PK와 같은 것이라고 설명한다. 테이블 전체가 PK 순서로 정리된 B-tree 하나라는 뜻이다. 용어집은 B-tree를 늘 정렬된 상태로 유지되는 구조라고 적고 있다.

그래서 새 행이 **어느 페이지에 들어가는지**는 PK 값이 정한다.

## 2. AUTO_INCREMENT PK는 늘 맨 끝에 붙는다

`AUTO_INCREMENT` PK는 새 행마다 앞 번호보다 큰 번호를 준다. 새 행의 PK가 늘 지금까지 나온 값 중 가장 크니, 들어갈 자리는 언제나 마지막 페이지다. 마지막 페이지가 차면 새 페이지를 하나 열어 이어서 채우고, 앞쪽 페이지는 다시 건드리지 않는다.

공식 문서에 따르면 이렇게 순서대로(오름차순이든 내림차순이든) 넣으면 페이지가 약 15/16까지 찬다. 남은 1/16은 나중에 행이 길어지는 경우를 대비해 InnoDB가 일부러 비워 두는 자리다.

## 3. 순서 없는 UUID는 중간에 끼워 넣는다

RFC 9562는 UUIDv4를 진짜 난수나 의사 난수로 만드는 UUID라고 정의한다. 방금 만든 값이 앞서 나온 값보다 클지 작을지 알 수 없으니, 새 행이 들어갈 자리도 테이블 처음일 수도, 가운데일 수도, 끝일 수도 있다. RFC 9562는 이런 값을 두고 "인덱스 지역성이 나쁘다"고 쓴다. 연달아 만든 값이 인덱스 안에서 서로 가까이 있지 않아서 INSERT를 무작위 위치에 해야 한다는 뜻이다.

문제는 들어갈 페이지가 이미 꽉 차 있을 때다. 번호순으로 책을 꽂아 둔 책장을 떠올리면 쉽다. 가운데 칸이 꽉 찼는데 그 칸 번호에 맞는 새 책이 왔다. 순서를 지켜야 하니 빈 칸 아무 데나 꽂을 수 없다. 그 칸의 책 절반쯤을 새 칸으로 옮겨 자리를 만든 뒤에 꽂는다. 이 작업이 페이지 분할이다. MySQL 용어집은 채움률(fill factor)을 "페이지가 분할되기 전까지 인덱스 데이터가 차지하는 비율"이라고 설명한다.

막 나눈 두 칸은 둘 다 반쯤 비어 있다. 다음 책이 마침 그 근처 번호라면 빈자리를 채운다. 번호가 무작위라 그런 책이 언제 올지는 모른다. 그렇게 책장 곳곳에 반쯤 빈 칸이 남는다. 공식 문서도 무작위 순서로 넣으면 페이지가 1/2에서 15/16 사이로 찬다고 적고 있다.

| 넣는 순서 | 새 행이 들어가는 자리 | 페이지가 차는 정도 (공식 문서) |
| --- | --- | --- |
| 번호순 (`AUTO_INCREMENT`) | 늘 맨 끝 | 약 15/16 |
| 무작위 (UUIDv4) | 매번 다른 중간 자리 | 1/2 ~ 15/16 |

행 수가 같아도 페이지가 덜 차 있으면 페이지가 더 많이 필요하다. 용어집의 GUID 항목에도 연달아 만든 값이 오름차순이 아니어서 큰 InnoDB 테이블의 PK로는 효율적이지 않다는 설명이 있다.

## 4. 버퍼 풀에 없는 페이지는 디스크에서 읽어 온다

InnoDB는 디스크에서 읽은 페이지를 버퍼 풀에 올려 두고 다음부터는 메모리에서 꺼내 쓴다. 공식 문서는 버퍼 풀을 InnoDB가 접근한 테이블과 인덱스 데이터를 캐시하는 메인 메모리 영역이라고 설명한다. 행을 하나 넣으려면 그 행이 들어갈 페이지가 버퍼 풀에 있어야 한다.

버퍼 풀은 책상, 디스크는 멀리 있는 서고라고 보면 된다. 번호순이면 늘 마지막 칸만 쓰니 그 칸이 계속 책상 위에 있다. 무작위면 이번엔 앞쪽 칸, 다음엔 가운데 칸이 필요하다. 테이블이 버퍼 풀보다 커지면 필요한 칸이 책상에 없을 때가 많아지고, 그때마다 서고까지 가서 가져와야 한다.

3번에서 본 것처럼 무작위 PK는 페이지가 덜 차서 테이블 자체도 크다. 테이블은 커지고 넣을 자리는 흩어지니 버퍼 풀에 다 올려 두기가 그만큼 더 어렵다. MySQL은 버퍼 풀에서 찾지 못해 디스크에서 직접 읽은 횟수를 `Innodb_buffer_pool_reads` 상태 변수로 보여 준다. UUID PK로 바꾼 뒤 INSERT가 느려졌다면 이 값이 같이 늘었는지 보면 된다.

RFC 9562는 시간 순서로 정렬되는 값과 무작위 값으로 넣을 때 실제 환경에서 10배 이상 차이가 날 수 있다고 적고 있다.

## 5. 보조 인덱스마다 PK가 따라붙는다

보조 인덱스는 PK가 아닌 컬럼으로 정리한 목록이다. 예시 테이블의 `idx_user_id`는 `user_id` 순서로 정리되어 있다. 공식 문서에 따르면 보조 인덱스의 레코드마다 그 행의 PK 컬럼이 들어 있다. `user_id`로 찾은 다음, 옆에 적힌 PK로 클러스터드 인덱스에서 행을 찾아가기 때문이다. 보조 인덱스가 여러 개면 PK 값도 그만큼 여러 번 저장된다.

공식 문서도 PK가 길면 보조 인덱스가 공간을 더 쓰니 PK를 짧게 두는 편이 낫다고 적고 있다.

PK 값 하나의 크기를 비교하면 다음과 같다.

| PK 타입 | 값 하나의 크기 | 근거 |
| --- | --- | --- |
| `BIGINT` | 8바이트 | 숫자 타입 저장 크기 표 |
| `BINARY(16)` | 16바이트 | `BINARY(M)`은 M바이트 |
| `CHAR(36)` (utf8mb4) | 36바이트 | COMPACT 계열 행 형식은 `CHAR(N)`에 최소 N바이트를 잡는다. UUID 문자열은 16진수 숫자와 하이픈뿐이라 한 글자가 1바이트다 |

예시 테이블처럼 보조 인덱스가 2개이고 행이 100만 개라면, 보조 인덱스 안에 들어가는 PK 값만 곱해 봐도 차이가 난다. 레코드 헤더나 페이지 빈자리는 뺀 단순 계산이다.

| PK 타입 | 계산 | 보조 인덱스 2개에 들어가는 PK 값 |
| --- | --- | --- |
| `BIGINT` | 8바이트 × 100만 × 2 | 1,600만 바이트 |
| `BINARY(16)` | 16바이트 × 100만 × 2 | 3,200만 바이트 |
| `CHAR(36)` | 36바이트 × 100만 × 2 | 7,200만 바이트 |

보조 인덱스도 인덱스라서 3번과 4번의 이야기가 그대로 적용된다. 레코드가 커지면 페이지 하나에 들어가는 레코드 수가 줄고, 인덱스가 커지면 버퍼 풀에 다 올리기 어려워진다.

## 6. MySQL의 UUID()는 왜 그대로 쓰면 순서가 어긋날까

MySQL에는 `UUID()` 함수가 있다. 공식 문서에 따르면 이 함수는 RFC 4122의 **버전 1** UUID를 `aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee` 모양 문자열로 돌려준다. 버전 1은 시각으로 만드는 UUID다. 앞의 세 덩어리가 타임스탬프의 low, middle, high 부분이다.

시계에 빗대 보자. 3시 5분 10초를 `10초-05분-03시`처럼 초부터 거꾸로 적어 둔 쪽지가 있다. 이런 쪽지들을 앞 글자부터 차례로 정렬하면 시간 순서가 되지 않는다.

| 실제 시각 | 초-분-시로 적은 쪽지 | 쪽지를 정렬한 순서 |
| --- | --- | --- |
| 3시 1분 20초 | `20-01-03` | 2번째 |
| 3시 5분 10초 | `10-05-03` | 1번째 |

더 늦은 시각인 3시 5분 10초가 정렬하면 앞에 온다. `UUID()`도 가장 빨리 바뀌는 부분(time-low)이 맨 앞에 있어서 문자열로 정렬한 순서와 만든 순서가 어긋난다.

`UUID_TO_BIN(uuid, swap_flag)`가 이 문제를 고친다. 문자열 UUID를 16바이트 이진값으로 바꾼다. `swap_flag`를 1로 주면 첫 번째 덩어리(time-low)와 세 번째 덩어리(time-high)의 자리를 바꾼다. 초-분-시로 적었던 쪽지를 시-분-초로 다시 적는 것과 같다. 공식 문서는 이렇게 하면 빨리 바뀌는 부분이 오른쪽으로 가서 인덱스 효율이 좋아질 수 있다고 설명한다.

```sql
SET @uuid = '6ccd780c-baba-1026-9564-5b8c656024db';

SELECT HEX(UUID_TO_BIN(@uuid, 0));  -- 6CCD780CBABA102695645B8C656024DB
SELECT HEX(UUID_TO_BIN(@uuid, 1));  -- 1026BABA6CCD780C95645B8C656024DB
```

공식 문서에 나온 예시다. `6CCD780C`와 `1026`의 자리가 바뀌었고 가운데 `BABA`는 그대로다.

테이블에서는 이렇게 쓴다. 넣을 때와 읽을 때 같은 `swap_flag`를 줘야 원래 문자열로 돌아온다.

```sql
CREATE TABLE orders_bin (
  id         BINARY(16) PRIMARY KEY,
  user_id    INT NOT NULL,
  amount     INT NOT NULL,
  created_at DATETIME NOT NULL,
  KEY idx_user_id (user_id),
  KEY idx_created_at (created_at)
) ENGINE=InnoDB;

-- 넣을 때: 시각 부분을 앞으로 옮겨서 16바이트로 저장
INSERT INTO orders_bin (id, user_id, amount, created_at)
VALUES (UUID_TO_BIN(UUID(), 1), 42, 15000, NOW());

-- 읽을 때: 같은 swap_flag로 원래 문자열로 되돌린다
SELECT BIN_TO_UUID(id, 1) AS id, user_id, amount FROM orders_bin;
```

> 공식 문서는 시각 부분 교환이 `UUID()`가 만드는 **버전 1** 값을 전제로 하며 버전 1 형식이 아닌 값에는 이득이 없다고 적고 있다. 애플리케이션에서 만든 UUIDv4나 UUIDv7에는 `swap_flag`를 쓰지 않는다.
{: .prompt-warning }

## 7. UUIDv7은 애플리케이션에서 만든다

RFC 9562는 UUIDv7을 새로 정의했다. 맨 앞 48비트에 밀리초 단위 Unix 타임스탬프를 넣고 버전 비트와 변형 비트를 뺀 나머지를 무작위 비트로 채운다. 시각이 맨 앞에 오니 나중에 만든 값이 대체로 더 크다. RFC 9562는 시간 순서 UUID를 쓰면 새 값끼리 인덱스 안에서 가까이 붙어 인덱스 지역성이 좋아진다고 설명한다. 같은 밀리초 안에 만든 값끼리는 뒤쪽이 무작위라 순서가 섞일 수 있다. 섞이는 범위는 그 밀리초 안으로 좁다.

MySQL 8.4에는 UUIDv7을 만드는 함수가 없다. 공식 문서의 UUID 관련 함수 중 값을 만드는 함수는 버전 1을 돌려주는 `UUID()`와, RFC 형식이 아닌 64비트 정수를 돌려주는 `UUID_SHORT()` 두 개뿐이다. UUIDv4를 만드는 함수도 없다. 그래서 UUIDv7은 애플리케이션에서 만들어 `BINARY(16)`으로 넣는다.

UUIDv7에 `swap_flag`를 주면 오히려 순서가 망가진다. 세 번째 덩어리에는 버전 번호와 무작위 비트가 들어 있어서 이걸 맨 앞으로 옮기면 무작위 값이 정렬의 첫 기준이 된다.

## 8. 해결 방법 비교

| 방법 | PK 값 크기 | 새 행이 들어가는 자리 | 좋은 점 | 주의할 점 |
| --- | --- | --- | --- | --- |
| `CHAR(36)`에 UUIDv4 (예시 테이블 2) | 36바이트 | 매번 무작위 | 사람이 읽기 쉽다 | 가장 크고, 페이지 분할과 디스크 읽기가 잦아진다 |
| `BIGINT AUTO_INCREMENT` | 8바이트 | 늘 맨 끝 | 가장 작고 순서대로 쌓인다 | 번호를 데이터베이스가 정한다. 그대로 외부에 보이면 1, 2, 3 순번이 드러난다 |
| `BINARY(16)`에 UUIDv4 | 16바이트 | 매번 무작위 | `CHAR(36)`보다 값이 절반 이하로 작다 | 순서 문제는 그대로 남는다 |
| `BINARY(16)` + `UUID_TO_BIN(UUID(), 1)` | 16바이트 | 시간 순서에 가깝다 | MySQL 내장 함수만으로 된다 | 버전 1 전용이다. 읽을 때 `BIN_TO_UUID(id, 1)`을 써야 한다 |
| `BINARY(16)`에 UUIDv7 | 16바이트 | 시간 순서에 가깝다. 같은 밀리초 안에서만 섞인다 | 표준 형식이고 애플리케이션 어디서 만들어도 시간순이다 | MySQL에 생성 함수가 없어 애플리케이션에서 만든다. `swap_flag`는 쓰지 않는다 |
| 내부 PK는 `BIGINT`, 외부용 UUID 컬럼을 따로 | PK 8바이트 | PK는 늘 맨 끝 | 보조 인덱스에는 8바이트 PK만 붙는다 | UUID 컬럼에 `UNIQUE` 인덱스가 하나 더 생긴다. UUIDv4를 넣으면 이 인덱스 하나는 3번 문제를 그대로 겪는다 |

마지막 방법은 이런 모양이다.

```sql
CREATE TABLE orders (
  id         BIGINT AUTO_INCREMENT PRIMARY KEY,  -- 테이블 안에서만 쓰는 번호
  public_id  BINARY(16) NOT NULL,                -- URL이나 API에 보여 주는 번호
  user_id    INT NOT NULL,
  amount     INT NOT NULL,
  created_at DATETIME NOT NULL,
  UNIQUE KEY uk_public_id (public_id),
  KEY idx_user_id (user_id),
  KEY idx_created_at (created_at)
) ENGINE=InnoDB;
```

외부 번호로 조회할 때는 `uk_public_id`에서 `id`를 찾은 뒤 클러스터드 인덱스에서 행을 읽는다.

## 정리

- InnoDB는 행을 PK 순서대로 저장한다. PK가 늘 커지면 맨 끝에 붙이고 순서가 없으면 중간에 끼워 넣는다.
- 중간에 끼워 넣다 보면 꽉 찬 페이지가 나뉘고 반쯤 빈 페이지가 남는다. 페이지가 많아지면 버퍼 풀에 다 올리기 어려워져 디스크 읽기가 늘어난다.
- PK 값은 모든 보조 인덱스에 다시 들어간다. PK가 크면 보조 인덱스가 모두 커진다.

UUID PK를 쓰는 테이블이 느리다면 다음 순서로 확인한다.

1. PK 타입이 `CHAR(36)`이나 `VARCHAR(36)`이라면 `BINARY(16)`으로 바꿀 수 있는지 본다.
2. UUID가 어디서 만들어지는지 본다. MySQL `UUID()`라면 `UUID_TO_BIN(uuid, 1)`, 애플리케이션이라면 UUIDv4 대신 UUIDv7을 쓸 수 있는지 본다.
3. 보조 인덱스가 몇 개인지 센다. 많을수록 PK 크기의 영향이 커진다.
4. UUID가 정말 PK여야 하는지 따져 본다. 외부에 보여 줄 번호만 필요하다면 내부 PK는 `BIGINT AUTO_INCREMENT`로 두고 UUID 컬럼을 따로 둔다.

## 참고 문서

- [MySQL 8.4 Reference Manual: Clustered and Secondary Indexes](https://dev.mysql.com/doc/refman/8.4/en/innodb-index-types.html)
- [MySQL 8.4 Reference Manual: The Physical Structure of an InnoDB Index](https://dev.mysql.com/doc/refman/8.4/en/innodb-physical-structure.html)
- [MySQL 8.4 Reference Manual: Buffer Pool](https://dev.mysql.com/doc/refman/8.4/en/innodb-buffer-pool.html)
- [MySQL 8.4 Reference Manual: InnoDB Row Formats](https://dev.mysql.com/doc/refman/8.4/en/innodb-row-format.html)
- [MySQL 8.4 Reference Manual: Data Type Storage Requirements](https://dev.mysql.com/doc/refman/8.4/en/storage-requirements.html)
- [MySQL 8.4 Reference Manual: Miscellaneous Functions (UUID, UUID_TO_BIN, BIN_TO_UUID)](https://dev.mysql.com/doc/refman/8.4/en/miscellaneous-functions.html)
- [MySQL 8.4 Reference Manual: Server Status Variables (Innodb_buffer_pool_reads)](https://dev.mysql.com/doc/refman/8.4/en/server-status-variables.html#statvar_Innodb_buffer_pool_reads)
- [MySQL 8.4 Reference Manual: MySQL Glossary](https://dev.mysql.com/doc/refman/8.4/en/glossary.html)
- [RFC 9562: Universally Unique IDentifiers (UUIDs)](https://www.rfc-editor.org/rfc/rfc9562.html)

