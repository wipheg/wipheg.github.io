---
title: "Why is COUNT(*) slow in InnoDB?"
date: 2026-11-10 06:00:00 +0900
categories: [Database, MySQL]
tags: [mysql, innodb, mvcc, index, count]
description: A query that returns a single number took over 3 seconds on a million rows. I ran a few experiments to see how InnoDB counts rows.
toc: true
lang: en
ref: innodb-count
hidden: true
---

```sql
SELECT COUNT(*) FROM posts;
```

The result is one number. On a table with a million posts, this query still took more than 3 seconds. After I added one index to the same table, it dropped to 0.1 seconds.

I read the MySQL documentation and ran some experiments to find out why. This post only covers `COUNT(*)` without `WHERE` or `GROUP BY`.

> **Summary**
> - InnoDB doesn't store a table's row count anywhere. It counts the rows every time you run `COUNT(*)`.
> - To count, it picks the smallest index on the table and reads it from start to finish.
> - On a 1-million-row table, counting through a 19.5MB index took 0.1 seconds. Counting through the whole 476MB table took 3.3 seconds.
> - `COUNT(1)` is the same as `COUNT(*)`. `COUNT(column)` can return a different number.
{: .prompt-tip }

## A few words first

Throughout this post, I'll compare counting rows to counting the students in a class.

| Term | In one line | In the classroom |
| --- | --- | --- |
| Row | One record in a table. Here, one post | One student |
| Transaction | A group of queries handled as one unit | One person's homework assignment |
| Index | A separately sorted copy of some data, built to find things fast | The index at the back of a book, or a class roster |
| Buffer pool | Memory where InnoDB keeps data it uses often | Your desk |
| Execution plan (`EXPLAIN`) | A preview of the steps MySQL will take to run a query | The teacher's to-do list on the board |

## Test setup

| Item | Value |
| --- | --- |
| MySQL | 8.4.13 (official Docker image) |
| Isolation level | `REPEATABLE READ` (default) |
| Buffer pool | 128MB (default) |
| Data | 1 million rows in `posts`, clustered index about 476MB |

```sql
CREATE TABLE posts (
  id         BIGINT AUTO_INCREMENT PRIMARY KEY,
  user_id    INT NOT NULL,
  title      VARCHAR(200) NOT NULL,
  body       TEXT NOT NULL,          -- filled with 400 characters to make each row large
  deleted_at DATETIME NULL,          -- set on only 1 in 10 rows
  created_at DATETIME NOT NULL
) ENGINE=InnoDB;
```

Each timing is the same query run three times. The absolute numbers will differ on your machine, so look at how many times faster or slower things get.

## 1. Why not just store the count?

Imagine a sign on the classroom door that says "30 students." If someone updates it every time a student comes in or leaves, you can answer "how many?" just by reading the sign. MyISAM, another MySQL storage engine, does exactly this. InnoDB has no such sign.

That's because two people can need two different answers at the same time. Here's an example from a message board.

- Admin A starts a transaction to build a monthly report and counts the posts: 1 million.
- While A pulls the other numbers for the report, user B writes 3 new posts.
- A counts the posts again inside the same transaction.

A should still get 1 million on the second count. If the number changed halfway through, the tables in the report wouldn't agree with each other. B, who just posted, should get 1,000,003. At the same moment, there are two correct answers.

That's what happened when I tried it. A counted the rows and waited 4 seconds, and in the meantime B inserted 3 rows and committed.

```text
[A] START TRANSACTION;
[A] SELECT COUNT(*) FROM posts;   -- 1000000
        [B] INSERT ... 3 rows;  (autocommit)
        [B] SELECT COUNT(*) FROM posts;   -- 1000003
[A] SELECT COUNT(*) FROM posts;   -- 1000000  <- A can't see B's new rows
[A] COMMIT;
[A] SELECT COUNT(*) FROM posts;   -- 1000003
```

This is how InnoDB behaves under `REPEATABLE READ`, the default isolation level. When A runs its first `SELECT` in the transaction, picture A taking a photo of the classroom. If B walks in afterward, B isn't in A's photo. A keeps looking at that same photo until the transaction ends. This approach is called MVCC (multi-version concurrency control).

With only one sign on the door, either A's photo or B's present would get the wrong number. So InnoDB skips the sign. Each `COUNT(*)` counts the students in the photo held by whoever is asking. The MySQL manual gives the same reason:

> InnoDB does not keep an internal count of rows in a table because concurrent transactions might "see" different numbers of rows at the same time.
>
> Source: [MySQL 8.4 Reference Manual, COUNT()](https://dev.mysql.com/doc/refman/8.4/en/aggregate-functions.html#function_count)

## 2. So InnoDB counts every time

There are two ways to count the students in a class. You can flip through a thick binder with a full file on every student, or through a thin class roster with just their names. You get the same answer either way, and the roster is much faster.

InnoDB has both:

| Classroom | InnoDB | In the test table |
| --- | --- | --- |
| Binder of student files | Clustered index. The table data itself, sorted by primary key | `PRIMARY`. Holds every full row |
| Class roster | Secondary index. A separate list with just some column values and the primary key | `idx_user_id`. Holds only `user_id` and `id` |

According to the manual, InnoDB handles `COUNT(*)` by reading the smallest secondary index, the thinnest roster, from start to finish. If there's no roster, it goes through the binder.

At first this table had no roster. The execution plan shows `key: PRIMARY`, meaning it will flip through the binder:

```text
         type: index
          key: PRIMARY
         rows: 945547
        Extra: Using index
```

`EXPLAIN FORMAT=TREE` shows a one-line plan:

```text
-> Count rows in posts
```

The plan just says "count rows." To do that, InnoDB flips through all 1 million entries from the first to the last.

## 3. Counting with a roster was 30 times faster

I built a roster (a secondary index) on `user_id` and compared the sizes.

```sql
ALTER TABLE posts ADD INDEX idx_user_id (user_id);
```

| Index | Size | Each entry holds |
| --- | --- | --- |
| `PRIMARY` (binder) | about 476MB | the full row, including `title` and the 400-character `body` |
| `idx_user_id` (roster) | about 19.5MB | `user_id` + primary key `id` |

Once the roster existed, MySQL picked the thinner one on its own:

```text
          key: idx_user_id
        Extra: Using index
```

I ran the same `COUNT(*)` and changed only which one it counted from.

| Index read | Run 1 | Run 2 | Run 3 |
| --- | --- | --- | --- |
| `PRIMARY` (no roster) | 3.30s | 3.30s | 3.26s |
| `idx_user_id` (chosen automatically) | 0.14s | 0.11s | 0.09s |
| `PRIMARY` (binder forced with `FORCE INDEX`) | 3.40s | 3.37s | 3.35s |

The gap comes from how many sheets there are to turn. InnoDB reads data in 16KB pages. Think of each page as one sheet in a notebook.

- The binder (`PRIMARY`) is 476MB, or about 30,000 sheets. Every entry carries a 400-character body, so only a few dozen students fit on a sheet.
- The roster (`idx_user_id`) is 19.5MB, or about 1,200 sheets. Each entry is just two numbers, so hundreds of students fit on a sheet.

Both count the same 1 million students, but one has about 25 times as many sheets to turn.

Desk size matters too. Think of the buffer pool as your desk and the disk as a storage room down the hall.

- The desk (buffer pool) is 128MB.
- The 19.5MB roster fits on the desk in one piece. From the second run on, nothing had to come from the storage room, and each count finished in about 0.1 seconds.
- The 476MB binder doesn't fit. Partway through, the desk fills up and the earliest sheets go back to the storage room. The next count has to fetch them again, which is why it stayed above 3 seconds no matter how many times I ran it.

> If a table has large columns like post bodies or JSON, the binder gets thicker while the roster stays thin. On tables like that, one small secondary index can change `COUNT(*)` speed a lot.
{: .prompt-info }

## 4. Even with a roster, it slows down as data grows

A roster helps when you know where to look, like "find student number 3." It lets you open just that page. In the test table, `WHERE user_id = 42` works that way. A full headcount has no such spot to jump to. However thin the roster is, you still turn every sheet from the first to the last. With 2 million students there are twice as many sheets, and the count takes about twice as long.

Sometimes the roster alone isn't enough. A roster lists names, but it doesn't say who changed an entry or when. Only the binder has that. According to the manual, InnoDB goes back to the binder in two cases:

- The name on the roster has a "to be removed" mark (a delete mark) next to it. It's like a pencil line through the name of a student who transferred out.
- Another transaction changed that roster page after A took the photo.

The binder records the number of the transaction that last changed each student's file. If that change happened after A's photo, InnoDB pulls out a separate stack of "before" notes kept next to the binder (the undo log) and checks whether the student was in A's photo. On a table where posts are constantly written and deleted, these checks pile up, and the count can take longer than the 0.1 seconds from this test.

## 5. Is COUNT(1) faster?

It's a common tip, and it does nothing in InnoDB. The manual says InnoDB handles `COUNT(*)` and `COUNT(1)` the same way, with no performance difference. In my test, both produced the plan `Count rows in posts` and both took around 0.09 seconds.

The `*` in `COUNT(*)` means "count rows." People sometimes read it like the `*` in `SELECT *`, as if it pulled every column, but it has nothing to do with columns.

`COUNT(column)` counts something else. In the classroom, `COUNT(*)` is the number of students, and `COUNT(homework)` is the number of students whose homework box isn't empty. Anyone who didn't turn in homework is left out.

```sql
SELECT COUNT(*), COUNT(1), COUNT(deleted_at) FROM posts;
```

```text
COUNT(*)  COUNT(1)  COUNT(deleted_at)
1000000   1000000   100000
```

| Expression | What it counts | Result | Plan | Time |
| --- | --- | --- | --- | --- |
| `COUNT(*)` | rows | 1,000,000 | `Count rows in posts` | about 0.1s |
| `COUNT(1)` | rows | 1,000,000 | `Count rows in posts` | about 0.1s |
| `COUNT(deleted_at)` | rows where `deleted_at` isn't empty (not `NULL`) | 100,000 | `Table scan on posts` | about 3.4s |

`deleted_at` is the time a post was deleted, so only deleted posts have a value. That means `COUNT(deleted_at)` counted deleted posts, which is why it returned 100,000. And because `deleted_at` isn't on the roster, InnoDB went through the whole 476MB binder to check each value.

If you want the number of rows, use `COUNT(*)`.

## 6. A rough number is available instantly

You can look at a crowd in the schoolyard and guess "about 95 kids." MySQL keeps a similar guess for each table's size, calculated ahead of time so it can plan queries.

```sql
SHOW TABLE STATUS LIKE 'posts'\G
```

```text
Engine: InnoDB
  Rows: 945547
```

The real count is 1,000,000, and this shows 945,547, a little over 5% low. The `rows: 945547` in the earlier `EXPLAIN` output is the same number. The manual describes it as a rough estimate used for optimization. Since it only reads a value that's already been calculated, it comes back immediately.

For a footer that says "about 940,000 posts," that's close enough.

## 7. Picking an approach

The right approach depends on where the number is used.

| Situation | Approach | Watch out for |
| --- | --- | --- |
| Number of my posts (`WHERE user_id = ?`) | Count just that user's range through the `user_id` index | The filtered column needs an index |
| Total post count on an admin page viewed a few times a day | Make sure there's a small secondary index and run `COUNT(*)` as is | It gets slower as data grows |
| Total post count shown on every home page request | A counter table or a cache | You need code that keeps the number correct |
| Showing an approximate figure like "about 940,000 posts" | The estimate from `SHOW TABLE STATUS` or `information_schema.TABLES` | It can be off by a few percent |

A counter table is the classroom sign from section 1, built by hand. Every time a post is written, the number goes up too.

```sql
CREATE TABLE table_counts (
  table_name VARCHAR(64) PRIMARY KEY,
  row_count  BIGINT NOT NULL
);

-- Update it in the same transaction that writes the post
START TRANSACTION;
INSERT INTO posts (user_id, title, body, created_at) VALUES (1, 'New post', '...', NOW());
UPDATE table_counts SET row_count = row_count + 1 WHERE table_name = 'posts';
COMMIT;
```

Reading the count means reading one sign, so it's fast. There are two things to keep track of, though.

- Every code path that changes the row count, including deletes, restores, and bulk imports, has to update the sign too. Miss one and the number drifts.
- There's one sign, and every post write changes it. Picture a line of students waiting to change a single number on the board. When many posts come in at once, that line gets long and writes slow down. The manual also warns that this may not scale when thousands of concurrent transactions update the same counter.

## Wrap-up

- InnoDB has no "row count" sign. Each transaction may be looking at a different photo, so `COUNT(*)` counts every time.
- It counts by turning every sheet of the thinnest roster (secondary index). In my test, that took 0.1 seconds with a 19.5MB roster and 3.3 seconds with the 476MB binder.
- It's fast when the roster fits on the desk (buffer pool), and slower on tables that change often because there's more to double-check.

When you run into a slow `COUNT(*)`, check these in order:

1. Run `EXPLAIN` and see whether `key` is `PRIMARY`. If it is, check whether a small secondary index exists.
2. If you're using `COUNT(column)`, check whether you really need to skip `NULL`s. If not, switch to `COUNT(*)`.
3. Ask whether the number has to be exact every time. If it doesn't, a cache or the estimate is enough.

## References

- [MySQL 8.4 Reference Manual: Aggregate Function Descriptions (COUNT)](https://dev.mysql.com/doc/refman/8.4/en/aggregate-functions.html)
- [MySQL 8.4 Reference Manual: Consistent Nonlocking Reads](https://dev.mysql.com/doc/refman/8.4/en/innodb-consistent-read.html)
- [MySQL 8.4 Reference Manual: InnoDB Multi-Versioning](https://dev.mysql.com/doc/refman/8.4/en/innodb-multi-versioning.html)
- [MySQL 8.4 Reference Manual: InnoDB Restrictions and Limitations](https://dev.mysql.com/doc/refman/8.4/en/innodb-restrictions-limitations.html)
