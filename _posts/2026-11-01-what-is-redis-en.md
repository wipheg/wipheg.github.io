---
title: "What Is Redis?"
date: 2026-11-01 09:00:00 +0900
categories: [Database, Redis]
tags: [redis, cache, in-memory, nosql]
description: Where and how Redis stores data, which data structures it offers, what happens when memory fills up or the server goes down, and what it is used for in practice.
toc: true
lang: en
ref: what-is-redis
hidden: true
---

Backend architecture diagrams often show Redis sitting next to the database. You hear "we use it as a cache" or "we put sessions there" all the time, but where Redis keeps its data, and whether that data survives a restart, takes some digging to find out. This post walks through how Redis stores data, deletes it, and writes it to disk, in that order.

> **Summary**
> - Redis is a key-value store that keeps all of its data in memory (RAM). Reads and writes do not touch the disk.
> - A value can be a string or a data structure such as a List, Hash, Set, or Sorted Set, and each structure has its own commands.
> - Commands are mostly executed one at a time by a single thread. So one slow command (`KEYS *` and the like) makes every request that arrives in the meantime wait.
> - With the default settings, when memory is full Redis does not delete keys and returns an error to write commands (`noeviction`). To use Redis as a cache, you have to change `maxmemory-policy`.
> - There are two ways to save to disk: RDB (periodic snapshots) and AOF (a log of write commands). AOF is off by default.
{: .prompt-tip }

## Terms to know first

| Term | Meaning |
| --- | --- |
| In-memory | Reading and writing data in RAM instead of on disk |
| Key-value store | A store that finds one value by one key. No SQL, no tables |
| TTL | Time left until a key is deleted automatically |
| Persistence | Data still being there after the server restarts |
| Eviction | Picking existing keys and deleting them when memory runs short |

## 1. What Redis is

Redis is a key-value store that keeps data in memory. Clients connect over TCP (default port 6379) and send commands such as `SET` and `GET`.

```text
127.0.0.1:6379> SET user:42:name "kim"
OK
127.0.0.1:6379> GET user:42:name
"kim"
```

It looks up data differently from a relational database such as MySQL.

| Item | MySQL (InnoDB) | Redis |
| --- | --- | --- |
| Where data lives | Disk, hot pages cached in memory | All in memory |
| How you query | SQL, indexes, joins | By key only |
| Size limit | Disk capacity | Memory capacity |
| After restart | Data is kept | Depends on disk persistence settings |

Redis has no conditional search like "users aged 30 or older". To get a value, you have to know its key. That is why teams usually settle on key names in a `type:ID:field` shape, like `user:42:name`, and build the keys in code.

## 2. A single thread runs commands one at a time

Even with thousands of clients connected, command execution is mostly handled by one main thread. Commands run one after another, so two clients running `INCR` on the same key at the same moment do not corrupt the value, and you do not need a lock.

```text
Client A ── INCR counter ─┐
Client B ── INCR counter ─┼──▶ [main thread] runs one at a time
Client C ── GET counter  ─┘        counter: 1 → 2 → read
```

The flip side is that while one command takes a long time, every other command waits. The most common example is `KEYS`. `KEYS` is an O(N) command whose time grows with the number of keys (N), so running `KEYS *` on a server with millions of keys stalls every other request until it finishes.

> The official docs say not to use `KEYS` in regular application code. If you need a list of keys, use `SCAN`, which fetches them a little at a time with a cursor. For the same reason, be careful with `HGETALL` and `SMEMBERS`, which fetch every element of a large Hash or Set at once.
{: .prompt-warning }

> Since Redis 6.0 there is an `io-threads` setting that hands client socket reads and writes, and request parsing, to separate I/O threads. It is off by default, so only the main thread is used.
{: .prompt-info }

## 3. Data structures

Besides strings, Redis stores several data structures, and each one has its own commands. The application changes the value inside Redis directly instead of reading it, modifying it, and writing it back.

| Type | Holds | Main commands | Used for |
| --- | --- | --- | --- |
| String | Text, numbers | `SET` `GET` `INCR` | Cache, counters |
| List | Ordered list | `LPUSH` `RPOP` `LRANGE` | Job queues, recent items |
| Hash | Field-value pairs | `HSET` `HGET` | Objects |
| Set | Unique members | `SADD` `SISMEMBER` | Dedup checks, tags |
| Sorted Set | Members sorted by score | `ZADD` `ZRANGE` | Rankings |

A single String value can hold up to 512MB. There are also Stream (log-style messages), Bitmap, HyperLogLog (approximate counting), and Geo (locations) types.

For example, a game leaderboard is built with one Sorted Set. Redis keeps the set sorted as scores are added, so showing the top 3 does not require sorting everything again.

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

## 4. Expiration (TTL)

When you set an expiration on a key, Redis deletes the key once that time has passed. Caches and sessions use this because they are valid only for a while.

```text
127.0.0.1:6379> SET session:abc123 "user:42" EX 1800   # expires in 30 minutes
OK
127.0.0.1:6379> TTL session:abc123
(integer) 1799
```

Expired keys are deleted in two ways.

- **On access (passive).** When a client tries to read a key that has expired, Redis deletes it then and answers as if the key does not exist.
- **By periodic sampling (active).** Redis picks a random sample of keys that have an expiration, checks them, and deletes the expired ones.

So a key past its expiration can stay in memory for a short while if nobody reads it. It never shows up in responses.

## 5. When memory fills up

Because Redis keeps data only in memory, you set how much memory it may use with `maxmemory`. What happens when that limit is reached is decided by `maxmemory-policy`.

| Policy | What gets deleted |
| --- | --- |
| `noeviction` (default) | Nothing. Write commands return an error |
| `allkeys-lru` | Least recently used key among all keys |
| `allkeys-lfu` | Least frequently used key among all keys |
| `volatile-lru` | Least recently used key among keys with a TTL |
| `volatile-ttl` | Key with the nearest expiration among keys with a TTL |
| `allkeys-random` | A random key among all keys |

There are also `volatile-lfu` and `volatile-random`, and `allkeys-lrm` and `volatile-lrm` were added in Redis 8.6.

> With the default `noeviction`, once memory is full, write commands such as `SET` return an error starting with `-OOM`. For a cache, switch to `allkeys-lru` or `allkeys-lfu` so old cache entries get deleted and new ones fit. On the other hand, if you mix data that must not be deleted, such as sessions, with cache data in one Redis, the eviction policy can delete sessions too.
{: .prompt-warning }

## 6. Saving to disk (persistence)

Data in memory disappears when the process stops. Redis offers two ways to keep data on disk.

### 6-1. RDB: periodic snapshots

RDB saves the entire dataset at that moment into a single file (`dump.rdb`) at set intervals. The default rule saves when any one of these three conditions is met.

- At least 1 key changed within 3600 seconds (1 hour)
- At least 100 keys changed within 300 seconds (5 minutes)
- At least 10000 keys changed within 60 seconds

Being a single file makes it easy to back up and restore, and it loads faster than AOF on restart. The trade-off is that changes made after the last snapshot are lost if the server crashes. With the rule above, a server with few changes can lose up to an hour of data.

### 6-2. AOF: a log of write commands

AOF (Append Only File) appends every command that changes data to the end of a file. On restart, Redis replays these commands from the beginning to rebuild the data.

How often AOF is actually flushed to disk (fsync) is set by `appendfsync`. The default, and the value the official docs recommend, is `everysec` (once per second), and with it a crash loses at most one second of writes.

```conf
# redis.conf
appendonly yes          # default is no
appendfsync everysec    # default
```

### 6-3. Comparison

| Item | RDB | AOF |
| --- | --- | --- |
| What it saves | Full dataset at one point in time | Log of write commands |
| What you can lose | Everything since the last snapshot | About 1 second with `everysec` |
| File size | Smaller than AOF | Grows as commands pile up, rewritten periodically |
| Default | On | Off (`appendonly no`) |

For a cache that can be refilled from the database if lost, people sometimes use RDB only or turn both off. For data that lives only in Redis, such as sessions or queues, turn on AOF. The official docs say to use both if you need data safety comparable to PostgreSQL.

## 7. What it is used for

### 7-1. Cache

This is the most common use. Database query results are stored in Redis, and when the same request comes in, Redis answers instead of the database. The pattern where the application checks the cache first and, on a miss, reads the database and fills the cache is called cache-aside.

```text
Request ──▶ App ──① GET product:7 ──▶ Redis
             │                         │
             │◀── ② hit: respond now ──┘
             │
             ├── ③ miss: query DB ──▶ MySQL
             │
             └── ④ SET product:7 ... EX 600 ──▶ Redis
```

Always set a TTL on cache entries. Without one, the old value keeps being served after the database value changes, and unused keys keep piling up in memory.

### 7-2. Session store

With several servers, keeping sessions in each server's memory means a user gets logged out the moment a request goes to a server other than the one they logged in on. With sessions in Redis, every server reads the same session.

### 7-3. Rate limiting

Count each user's requests with `INCR`, and set the expiration on the first request. The example below allows up to 100 requests per minute.

```text
INCR   rate:user:42      # a result of 1 means this is the first request
EXPIRE rate:user:42 60   # set a 60-second expiration only on the first request
                         # reject the request if the INCR result exceeds 100
```

### 7-4. Rankings

As in the Sorted Set example in section 3, adding scores keeps the set sorted. `ZREVRANK` returns a given user's rank directly.

### 7-5. Distributed locks

To keep several servers from doing the same job at once, take a lock with `SET ... NX`, which stores the key only if it does not exist.

```text
SET lock:order:1001 "a7f3-uuid" NX PX 30000   # store only if absent, auto-release after 30s
```

> Do not release the lock with a plain `DEL`. If your lock already expired after 30 seconds and another server has taken a new one, you would delete their lock. Check that the value (`a7f3-uuid`) is the one you set and delete it in a single step, wrapped in a Lua script.
{: .prompt-warning }

## 8. Scaling out

| Setup | What it does |
| --- | --- |
| Replication | Replicas copy the master's data. Read scaling, backups |
| Sentinel | Detects master failure and promotes a replica to master |
| Cluster | Splits data across several masters |

Redis Cluster divides the key space into 16384 hash slots and assigns the slots to the masters. Which slot a key goes to is decided by `CRC16(key) mod 16384`.

> In a cluster, commands that touch several keys, like `MSET`, work only when all the keys are in the same slot. If you put `{}` (a hash tag) in the key name, only the string inside the braces is used to compute the slot. `{user1000}.following` and `{user1000}.followers` land in the same slot.
{: .prompt-warning }

## 9. Example: using it from Spring Boot

Add `spring-boot-starter-data-redis` and set the connection details.

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
            return cached;                      // return right away on a cache hit
        }

        String name = repository.findById(id)
                .orElseThrow()
                .getName();                     // on a miss, query the DB
        redis.opsForValue().set(key, name, Duration.ofMinutes(10)); // TTL 10 minutes
        return name;
    }
}
```

> Starting with Spring Boot 3, the property keys moved from `spring.redis.*` to `spring.data.redis.*`. If you copy `spring.redis.host` from an older blog example into a Spring Boot 3 app, it is ignored and the app connects to the default `localhost:6379`.
{: .prompt-warning }

## Wrap-up

- Redis is a key-value store that keeps all data in memory. You look things up by key only, and the data cannot exceed memory capacity.
- One main thread runs commands one at a time. `INCR` is safe without locks, but a single O(N) command like `KEYS *` stops every request while it runs.
- String, List, Hash, Set, and Sorted Set each have their own commands, so counters, queues, and rankings are handled inside Redis directly.
- The default (`noeviction`) refuses writes when memory is full. For a cache, switch to `allkeys-lru` or `allkeys-lfu`.
- By default only RDB snapshots are on, so a crash loses data written since the last snapshot. For data that lives only in Redis, turn on AOF (`everysec`, at most about one second lost).

Checklist before adopting Redis:

1. Is the data going into this Redis a cache that can be lost, or data that exists only in Redis?
2. Have you set `maxmemory` and `maxmemory-policy` for that use?
3. Do your cache keys have a TTL?
4. Is `KEYS` absent from your code?
5. Do the persistence settings (RDB, AOF) fit the use?

## References

- [Redis: Data types](https://redis.io/docs/latest/develop/data-types/)
- [Redis: Strings](https://redis.io/docs/latest/develop/data-types/strings/)
- [Redis: KEYS](https://redis.io/docs/latest/commands/keys/)
- [Redis: EXPIRE](https://redis.io/docs/latest/commands/expire/)
- [Redis: Key eviction](https://redis.io/docs/latest/develop/reference/eviction/)
- [Redis: Persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)
- [Redis Cluster specification](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/)
- [redis.conf (default configuration file)](https://raw.githubusercontent.com/redis/redis/unstable/redis.conf)
- [Spring Boot 3.0 Migration Guide: Redis Properties](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide)
- [Spring Boot: Working with NoSQL Technologies (Redis)](https://docs.spring.io/spring-boot/reference/data/nosql.html)
