---
title: "InnoDB에서 COUNT(*)는 왜 느릴까?"
date: 2026-11-10 06:00:00 +0900
categories: [Database, MySQL]
tags: [mysql, innodb, mvcc, index, count]
description: 숫자 하나를 돌려주는 쿼리가 100만 건에서 3초가 걸렸다. InnoDB가 행 개수를 세는 방식을 실험으로 따라가 봤다.
toc: true
lang: ko-KR
ref: innodb-count
---

```sql
SELECT COUNT(*) FROM posts;
```

결과는 숫자 하나다. 그런데 글이 100만 건인 테이블에서 이 쿼리를 돌려 보니 3초가 넘게 걸렸다. 같은 테이블에 인덱스 하나를 추가하자 0.1초로 줄었다.

왜 이런 차이가 나는지 MySQL 공식 문서를 읽고 직접 실험해 봤다. 여기서는 `WHERE`와 `GROUP BY`가 없는 `COUNT(*)`만 다룬다.

> **요약**
> - InnoDB는 테이블의 행 개수를 따로 적어 두지 않는다. `COUNT(*)`를 실행할 때마다 직접 센다.
> - 셀 때는 테이블에서 가장 작은 인덱스를 골라 처음부터 끝까지 읽는다.
> - 100만 건 테이블에서 19.5MB짜리 인덱스로 세면 0.1초, 476MB짜리 테이블 전체로 세면 3.3초가 걸렸다.
> - `COUNT(1)`은 `COUNT(*)`와 똑같다. `COUNT(컬럼)`은 결과가 달라질 수 있다.
{: .prompt-tip }

## 먼저 알아둘 단어

글 전체를 한 반의 학생 수를 세는 일에 빗대어 설명한다.

| 단어 | 한 줄 설명 | 교실에 빗대면 |
| --- | --- | --- |
| 행 | 테이블에 들어 있는 데이터 한 줄. 여기서는 글 하나 | 학생 한 명 |
| 트랜잭션 | 여러 쿼리를 한 덩어리로 묶어 처리하는 단위 | 한 사람이 맡은 숙제 한 건 |
| 인덱스 | 원하는 데이터를 빨리 찾으려고 따로 만든 정리본 | 책 뒤의 찾아보기, 반 출석부 |
| 버퍼 풀 | InnoDB가 자주 쓰는 데이터를 올려 두는 메모리 | 내 책상 |
| 실행 계획 (`EXPLAIN`) | MySQL이 쿼리를 어떤 순서로 처리할지 미리 보여 주는 계획표 | 선생님이 칠판에 적은 오늘 할 일 |

## 실험 환경

| 항목 | 값 |
| --- | --- |
| MySQL | 8.4.13 (Docker 공식 이미지) |
| 격리 수준 | `REPEATABLE READ` (기본값) |
| 버퍼 풀 | 128MB (기본값) |
| 데이터 | `posts` 테이블 100만 건, 클러스터드 인덱스 약 476MB |

```sql
CREATE TABLE posts (
  id         BIGINT AUTO_INCREMENT PRIMARY KEY,
  user_id    INT NOT NULL,
  title      VARCHAR(200) NOT NULL,
  body       TEXT NOT NULL,          -- 400자씩 채워서 행을 일부러 크게 만들었다
  deleted_at DATETIME NULL,          -- 10건 중 1건만 값이 있다
  created_at DATETIME NOT NULL
) ENGINE=InnoDB;
```

시간은 같은 쿼리를 세 번씩 실행한 값이다. 컴퓨터마다 절대값은 달라지니 **몇 배 차이가 나는지**만 보면 된다.

## 1. 왜 개수를 적어 두지 않을까

교실 문 앞에 "우리 반 30명"이라고 적힌 팻말이 있다고 해 보자. 학생이 들어오고 나갈 때마다 숫자를 고치면 언제든 팻말만 보고 답할 수 있다. MyISAM이라는 다른 저장 엔진은 실제로 이렇게 한다. 그런데 InnoDB는 이 팻말을 두지 않는다.

이유는 사람마다 봐야 하는 답이 다를 때가 있어서다. 게시판으로 예를 들어 보자.

- 관리자 A가 월간 보고서를 만들려고 트랜잭션을 시작하고 글 수를 센다. 100만 건이다.
- A가 보고서의 다른 숫자를 뽑는 동안 사용자 B가 글 3개를 올린다.
- A가 같은 트랜잭션 안에서 글 수를 다시 센다.

A는 다시 세도 100만 건을 받아야 한다. 보고서를 만드는 도중에 숫자가 바뀌면 표끼리 맞지 않는다. B는 방금 올린 글까지 더한 1,000,003건을 받아야 한다. 같은 순간에 정답이 두 개다.

실제로 돌려 본 결과도 그랬다. A가 개수를 센 뒤 4초 기다리는 동안 B가 3건을 넣고 커밋했다.

```text
[A] START TRANSACTION;
[A] SELECT COUNT(*) FROM posts;   -- 1000000
        [B] INSERT ... 3건;  (자동 커밋)
        [B] SELECT COUNT(*) FROM posts;   -- 1000003
[A] SELECT COUNT(*) FROM posts;   -- 1000000  ← B가 넣은 행이 안 보인다
[A] COMMIT;
[A] SELECT COUNT(*) FROM posts;   -- 1000003
```

InnoDB는 기본 격리 수준인 `REPEATABLE READ`에서 이렇게 동작한다. A가 트랜잭션 안에서 처음 `SELECT`를 하는 순간 교실 사진을 한 장 찍는다고 생각하면 된다. 그 뒤로 B가 교실에 들어와도 A가 들고 있는 사진에는 B가 없다. A는 트랜잭션이 끝날 때까지 그 사진만 본다. 이런 방식을 MVCC(다중 버전 동시성 제어)라고 부른다.

교실 문에 팻말이 하나뿐이면 A의 사진과 B의 현실 중 한쪽은 틀린 숫자를 보게 된다. 그래서 InnoDB는 팻말을 두지 않고, `COUNT(*)`를 실행한 사람이 보는 사진 속 학생을 그때그때 센다. 공식 문서도 같은 이유를 적고 있다.

> InnoDB does not keep an internal count of rows in a table because concurrent transactions might "see" different numbers of rows at the same time.
>
> — [MySQL 8.4 Reference Manual, COUNT()](https://dev.mysql.com/doc/refman/8.4/en/aggregate-functions.html#function_count)

## 2. 그래서 매번 직접 센다

반 학생 수를 세는 방법은 두 가지다. 학생마다 한 장씩 있는 두꺼운 생활기록부를 넘기며 세도 되고, 이름만 적힌 얇은 출석부를 넘기며 세도 된다. 답은 같고 출석부 쪽이 훨씬 빨리 끝난다.

InnoDB에도 이 두 가지가 있다.

| 교실 | InnoDB | 실험 테이블에서 |
| --- | --- | --- |
| 생활기록부 | 클러스터드 인덱스. 테이블 데이터 자체가 기본 키 순서로 정리되어 있다 | `PRIMARY`. 행 전체가 들어 있다 |
| 출석부 | 보조 인덱스. 컬럼 값과 기본 키만 따로 정리한 목록이다 | `idx_user_id`. `user_id`와 `id`만 들어 있다 |

공식 문서에 따르면 InnoDB는 `COUNT(*)`를 할 때 **가장 작은 보조 인덱스**, 즉 가장 얇은 출석부를 처음부터 끝까지 넘긴다. 출석부가 없으면 생활기록부를 넘긴다.

처음에는 이 테이블에 출석부가 없었다. 실행 계획을 보면 `key`가 `PRIMARY`, 즉 생활기록부를 넘기겠다고 나온다.

```text
         type: index
          key: PRIMARY
         rows: 945547
        Extra: Using index
```

`EXPLAIN FORMAT=TREE`로 보면 계획은 한 줄이다.

```text
-> Count rows in posts
```

계획표에는 "행을 센다"라고 한 줄만 적혀 있다. 이걸 하려고 InnoDB는 100만 장을 처음부터 끝까지 넘긴다.

## 3. 출석부로 세면 30배 빨랐다

`user_id` 컬럼으로 출석부(보조 인덱스)를 하나 만들고 크기를 비교했다.

```sql
ALTER TABLE posts ADD INDEX idx_user_id (user_id);
```

| 인덱스 | 크기 | 한 줄에 담긴 것 |
| --- | --- | --- |
| `PRIMARY` (생활기록부) | 약 476MB | 행 전체 (`title`, `body` 400자 포함) |
| `idx_user_id` (출석부) | 약 19.5MB | `user_id` + 기본 키 `id` |

출석부가 생기자 MySQL이 알아서 얇은 쪽을 골랐다.

```text
          key: idx_user_id
        Extra: Using index
```

같은 `COUNT(*)`를 어느 쪽으로 세는지만 바꿔 가며 실행했다.

| 읽은 인덱스 | 1회 | 2회 | 3회 |
| --- | --- | --- | --- |
| `PRIMARY` (출석부 없음) | 3.30초 | 3.30초 | 3.26초 |
| `idx_user_id` (자동 선택) | 0.14초 | 0.11초 | 0.09초 |
| `PRIMARY` (`FORCE INDEX`로 생활기록부를 강제) | 3.40초 | 3.37초 | 3.35초 |

차이가 이렇게 큰 이유는 넘겨야 하는 종이 수에 있다. InnoDB는 데이터를 16KB짜리 페이지 단위로 읽는다. 공책을 한 장씩 넘긴다고 생각하면 된다.

- 생활기록부(`PRIMARY`)는 476MB라서 공책 **약 3만 장**이다. 한 줄마다 400자짜리 본문이 들어 있어서 한 장에 학생 수십 명밖에 못 적는다.
- 출석부(`idx_user_id`)는 19.5MB라서 공책 **약 1,200장**이다. 한 줄에 숫자 두 개뿐이라 한 장에 수백 명이 들어간다.

두 쪽 다 학생 100만 명을 세지만, 넘겨야 하는 장 수가 25배쯤 차이 난다.

책상 크기도 영향을 준다. 버퍼 풀을 책상, 디스크를 멀리 있는 서고라고 생각해 보자.

- 책상(버퍼 풀)은 128MB다.
- 19.5MB짜리 출석부는 책상에 통째로 올려 둘 수 있다. 그래서 두 번째부터는 서고에 갈 필요 없이 0.1초 안에 끝났다.
- 476MB짜리 생활기록부는 책상에 다 올라가지 않는다. 넘기다 보면 책상이 꽉 차서 앞에서 본 장부터 서고로 돌려보내야 한다. 다음에 다시 세면 그 장들을 또 서고에서 가져와야 해서 몇 번을 돌려도 3초대였다.

> 행에 긴 본문이나 JSON 같은 큰 컬럼이 있으면 생활기록부는 더 두꺼워지고 출석부는 그대로 얇다. 이런 테이블일수록 작은 보조 인덱스 하나가 `COUNT(*)` 속도를 크게 바꾼다.
{: .prompt-info }

## 4. 출석부가 있어도 데이터가 늘면 같이 느려진다

출석부는 "3번 학생 찾기"처럼 찾을 자리가 정해져 있을 때 그 장만 펼쳐 보게 해 준다. 실험 테이블이라면 `WHERE user_id = 42`가 그런 경우다. 전체 학생 수는 찾을 자리가 따로 없어서 출석부가 아무리 얇아도 첫 장부터 마지막 장까지 넘겨야 한다. 학생이 200만 명이 되면 넘길 장도 두 배가 되고 시간도 그만큼 는다.

출석부만 보고 끝나지 않을 때도 있다. 출석부에는 이름만 있고 "누가 언제 고쳤는지"는 적혀 있지 않다. 그 기록은 생활기록부에만 있다. 공식 문서에 따르면 다음 두 경우에 InnoDB는 생활기록부를 다시 펼쳐서 확인한다.

- 출석부의 이름 옆에 "지워질 예정" 표시(delete-mark)가 붙어 있을 때. 전학 간 학생 이름에 연필로 줄을 그어 둔 것과 비슷하다.
- A가 사진을 찍은 뒤에 다른 트랜잭션이 그 출석부 장을 고쳤을 때

생활기록부를 펼치면 그 학생 기록을 마지막으로 고친 트랜잭션 번호가 적혀 있다. A의 사진보다 나중에 고쳐졌다면, 생활기록부 옆에 따로 모아 둔 "고치기 전 내용" 메모(undo 로그)를 꺼내 A의 사진에 이 학생이 있었는지 맞춰 본다. 글이 계속 올라오고 지워지는 테이블이라면 이런 확인이 늘어나서 실험의 0.1초보다 오래 걸릴 수 있다.

## 5. COUNT(1)로 바꾸면 빨라질까?

자주 듣는 팁인데 InnoDB에서는 효과가 없다. 공식 문서는 InnoDB가 `COUNT(*)`와 `COUNT(1)`을 같은 방식으로 처리하며 성능 차이가 없다고 명시한다. 실험에서도 실행 계획이 똑같이 `Count rows in posts`였고 시간도 0.09초 안팎으로 같았다.

`COUNT(*)`의 `*`는 "행을 센다"는 뜻이다. `SELECT *`처럼 모든 컬럼을 읽는다는 뜻으로 오해하기 쉬운데, 컬럼과는 상관이 없다.

`COUNT(컬럼)`은 세는 대상이 다르다. 교실로 치면 `COUNT(*)`는 반 학생 수이고, `COUNT(숙제)`는 숙제 칸이 비어 있지 않은 학생 수다. 숙제를 안 낸 학생은 빠진다.

```sql
SELECT COUNT(*), COUNT(1), COUNT(deleted_at) FROM posts;
```

```text
COUNT(*)  COUNT(1)  COUNT(deleted_at)
1000000   1000000   100000
```

| 표현 | 세는 것 | 결과 | 실행 계획 | 시간 |
| --- | --- | --- | --- | --- |
| `COUNT(*)` | 행 개수 | 1,000,000 | `Count rows in posts` | 약 0.1초 |
| `COUNT(1)` | 행 개수 | 1,000,000 | `Count rows in posts` | 약 0.1초 |
| `COUNT(deleted_at)` | `deleted_at`이 비어 있지 않은(`NULL`이 아닌) 행 개수 | 100,000 | `Table scan on posts` | 약 3.4초 |

`deleted_at`은 글을 지운 시각이라 값이 있는 행은 지워진 글뿐이다. 그래서 `COUNT(deleted_at)`은 사실상 **지워진 글의 수**를 센 것이고 결과가 10만으로 나왔다. 게다가 `deleted_at`은 출석부에 없는 칸이라 값을 보려고 476MB 생활기록부를 전부 넘겼다.

전체 행 개수가 필요하면 `COUNT(*)`를 쓰면 된다.

## 6. 대략적인 숫자는 바로 나온다

운동장에 모인 학생들을 보고 "대충 95명쯤"이라고 눈대중하는 것처럼, MySQL도 테이블 크기를 어림잡은 값을 갖고 있다. 쿼리 계획을 짤 때 쓰려고 미리 계산해 둔 값이다.

```sql
SHOW TABLE STATUS LIKE 'posts'\G
```

```text
Engine: InnoDB
  Rows: 945547
```

실제는 1,000,000건인데 945,547건으로 나왔다. 5% 남짓 적다. 앞의 `EXPLAIN` 결과에 나온 `rows: 945547`도 같은 값이다. 공식 문서도 이 값은 최적화에 쓰는 대략적인 추정치라고 설명한다. 이미 계산해 둔 값을 읽기만 하므로 바로 나온다.

게시판 하단에 "글 약 94만 개"처럼 보여 주는 용도라면 이 정도로 충분하다.

## 7. 상황별로 고르기

그 숫자를 어디에 쓰는지에 따라 방법이 달라진다.

| 상황 | 방법 | 주의할 점 |
| --- | --- | --- |
| 내 글 목록 개수 (`WHERE user_id = ?`) | `user_id` 인덱스로 해당 사용자 부분만 센다 | 조건 컬럼에 인덱스가 있어야 한다 |
| 관리자 화면에서 하루 몇 번 보는 전체 글 수 | 작은 보조 인덱스가 있는지 확인하고 그대로 `COUNT(*)` | 데이터가 늘면 시간도 는다 |
| 메인 화면에서 요청마다 보여 주는 전체 글 수 | 카운터 테이블이나 캐시 | 숫자를 맞게 유지하는 코드가 필요하다 |
| "글 약 94만 개"처럼 대략 보여 줄 때 | `SHOW TABLE STATUS`나 `information_schema.TABLES`의 추정치 | 수 % 차이가 난다 |

카운터 테이블은 1번에서 말한 교실 팻말을 직접 만드는 방법이다. 글을 쓸 때마다 숫자도 같이 올린다.

```sql
CREATE TABLE table_counts (
  table_name VARCHAR(64) PRIMARY KEY,
  row_count  BIGINT NOT NULL
);

-- 글을 쓸 때 같은 트랜잭션 안에서 함께 갱신한다
START TRANSACTION;
INSERT INTO posts (user_id, title, body, created_at) VALUES (1, '새 글', '...', NOW());
UPDATE table_counts SET row_count = row_count + 1 WHERE table_name = 'posts';
COMMIT;
```

조회는 팻말 하나만 보면 되니 빠르다. 대신 두 가지를 챙겨야 한다.

- 글 삭제, 복구, 일괄 등록처럼 행 수가 바뀌는 모든 코드에서 팻말 숫자도 같이 고쳐야 한다. 한 군데라도 빠뜨리면 숫자가 어긋난다.
- 팻말은 하나인데 모든 글쓰기가 그 숫자를 고친다. 학생 여러 명이 칠판의 숫자 하나를 고치려고 줄을 서는 모습과 같다. 동시에 글이 많이 올라오면 이 줄이 길어져서 글쓰기가 느려진다. 공식 문서도 수천 개의 트랜잭션이 같은 카운터를 동시에 고치면 잘 확장되지 않을 수 있다고 적고 있다.

## 정리

- InnoDB에는 "행 개수" 팻말이 없다. 트랜잭션마다 보는 사진이 달라서 `COUNT(*)`는 매번 직접 센다.
- 셀 때는 가장 얇은 출석부(보조 인덱스)를 처음부터 끝까지 넘긴다. 실험에서는 19.5MB 출석부로 0.1초, 476MB 생활기록부로 3.3초가 걸렸다.
- 출석부가 책상(버퍼 풀)에 다 올라가면 빠르고, 글이 자주 바뀌는 테이블이면 확인할 거리가 늘어 느려진다.

느린 `COUNT(*)`를 만나면 다음 순서로 확인한다.

1. `EXPLAIN`을 실행해 `key`가 `PRIMARY`인지 본다. 그렇다면 작은 보조 인덱스가 있는지 확인한다.
2. `COUNT(컬럼)`을 쓰고 있다면 정말 `NULL`을 빼고 세야 하는지 확인하고 아니면 `COUNT(*)`로 바꾼다.
3. 그 숫자가 매번 정확해야 하는지 따져 본다. 아니라면 캐시나 추정치로 충분하다.

## 참고 문서

- [MySQL 8.4 Reference Manual: Aggregate Function Descriptions (COUNT)](https://dev.mysql.com/doc/refman/8.4/en/aggregate-functions.html)
- [MySQL 8.4 Reference Manual: Consistent Nonlocking Reads](https://dev.mysql.com/doc/refman/8.4/en/innodb-consistent-read.html)
- [MySQL 8.4 Reference Manual: InnoDB Multi-Versioning](https://dev.mysql.com/doc/refman/8.4/en/innodb-multi-versioning.html)
- [MySQL 8.4 Reference Manual: InnoDB Restrictions and Limitations](https://dev.mysql.com/doc/refman/8.4/en/innodb-restrictions-limitations.html)
