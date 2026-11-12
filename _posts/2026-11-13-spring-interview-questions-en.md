---
title: "5 Spring Backend Interview Questions and Their Follow-ups"
date: 2026-11-13 00:30:00 +0900
categories: [Backend, Spring]
tags: [spring, spring-boot, jpa, transactional, interview]
description: Five questions that come up often in Spring backend interviews, with the follow-up questions that come after the first answer and what to cover in each answer.
toc: true
lang: en
ref: spring-interview-questions
hidden: true
---

Spring interviews are decided by the follow-up questions more than the first one. If you answer "How does `@Transactional` work?" with "It works through a proxy", the interviewer asks next, "Then what happens when you call it from inside the same class?" This post covers five common questions, each with a first answer and three or four follow-ups.

> **Summary**
> - Constructor injection fixes dependencies as `final` and turns circular references into a failure at application startup.
> - `@Transactional` works by a proxy intercepting the method call and opening a transaction. That is why it does not apply to calls inside the same class.
> - N+1 happens when associated entities are lazy-loaded: one query for the list plus one query per row. Fetch join or a batch size setting reduces it.
> - The persistence context keeps entities in the first-level cache during a transaction, finds changed fields right before commit, and builds UPDATE statements.
> - One Tomcat thread handles one request through Filter, DispatcherServlet, Interceptor and Controller in that order. The limits on threads and DB connections decide throughput.
{: .prompt-tip }

## Terms to know first

| Term | Meaning |
| --- | --- |
| Bean | An object the Spring container creates and manages |
| Proxy | An object that receives calls in place of the real object and adds work before and after |
| Persistence context | The space where JPA keeps entities and tracks their changes |
| Lazy loading | Fetching an associated entity with a query only when it is actually used |
| OSIV | Open Session In View. Keeps the persistence context open until the HTTP response |

## 1. Why use constructor injection?

### 1-1. First answer

Spring supports three injection styles: field injection, setter injection and constructor injection. Give three reasons for constructor injection.

- Dependencies can be declared as `final` fields. Once the object is created, they cannot be swapped for another implementation.
- In tests you can create the object without Spring, as `new OrderService(fakeRepository)`. If you leave out a dependency, the constructor arguments do not match and you get a compile error.
- Every dependency shows up in the constructor arguments. When the arguments grow to 7 or 8, treat it as a sign that one class is doing too much, and split it.

Field injection differs on the second point. If you create the object with `new` without Spring, it compiles, but the field stays `null`.

```java
// Creating a field-injected class directly in a test
OrderService service = new OrderService(); // no compile error
service.order(); // orderRepository is null, so NullPointerException here
```

```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;

    // With a single constructor, @Autowired can be omitted
    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
}
```

### 1-2. Follow-up questions

**Q. What happens if A injects B and B injects A through constructors?**

The application fails at startup with `BeanCurrentlyInCreationException`. Creating A needs B, and creating B needs A, which does not exist yet. Since Spring Boot 2.6, circular references are blocked by default even with field or setter injection. You can allow them with `spring.main.allow-circular-references=true`, but the usual fix is to move the shared logic into a third class to break the cycle.

For example, say `OrderService` injects `MemberService` to check a member's grade when placing an order, and `MemberService` injects `OrderService` to check for open orders before a member leaves. If what both services actually use is one thing, "look up a member's orders", move that into a `MemberOrderQuery` class. When both services inject only `MemberOrderQuery`, they no longer point at each other.

```text
Before: OrderService ──▶ MemberService ──▶ OrderService   (cycle)
After:  OrderService ──▶ MemberOrderQuery ◀── MemberService
```

**Q. If there are two beans of the same type, which one is injected?**

With no extra setting, you get `NoUniqueBeanDefinitionException`. There are three ways to fix it.

- `@Primary`: put it on the one bean to use by default.
- `@Qualifier("name")`: name the bean on the injecting side. It takes priority over `@Primary`.
- Inject all of them as `List<PaymentClient>` or `Map<String, PaymentClient>`. This is used for the strategy pattern, picking an implementation per payment method.

**Q. Beans are singletons by default. Does that cause any problems?**

Because the default scope is singleton, many request threads use one bean at the same time. If a bean stores a per-request value in an instance field, user B's request can read the value user A's request put there. Pass per-request values as local variables or method arguments.

```java
@Service
public class PriceService {
    private int discount; // every request thread shares this one field

    // Wrong: writes to the field, then reads it back
    public int calcWrong(int price, int discount) {
        this.discount = discount;
        return price - this.discount;
    }

    // Right: each thread has its own arguments
    public int calc(int price, int discount) {
        return price - discount;
    }
}
```

If two requests hit `calcWrong()` at almost the same time, this order can happen.

```text
1. Request A: this.discount = 1000
2. Request B: this.discount = 3000
3. Request A: computes price - this.discount → subtracts 3000, not 1000
```

## 2. How does `@Transactional` work?

### 2-1. First answer

Spring registers a proxy object in the container in place of the bean that has `@Transactional`. When another bean calls this service, the proxy receives the call first. The proxy starts a transaction, calls the real method, then commits on normal return and rolls back on an exception.

```text
Controller ──call──> [Proxy] ──> start transaction
                        │
                        ├──> OrderService.order() runs (real object)
                        │
                        └──> normal return: commit / RuntimeException: rollback
```

### 2-2. Follow-up questions

**Q. What if you call a `@Transactional` method from inside the same class?**

No transaction is applied. `this.save()` calls the real object's method directly without going through the proxy. This is called the self-invocation problem.

```java
@Service
public class OrderService {
    public void order() {
        save(); // this.save(): does not go through the proxy
    }

    @Transactional
    public void save() { ... } // no transaction is opened
}
```

The fix is to move `save()` to another bean. Putting `@Transactional` on the calling method `order()` also works.

**Q. Does a checked exception cause a rollback?**

With the default settings, it does not roll back. It commits. Spring treats only `RuntimeException` and `Error` as rollback triggers. To roll back on a checked exception such as `IOException`, specify it, as in `@Transactional(rollbackFor = Exception.class)`.

> If the method catches the exception with `try-catch` and does not rethrow it, the proxy never sees the exception. It treats the call as a normal return and commits.
{: .prompt-warning }

**Q. The order fails, but the failure log must still be saved. How?**

Move the log-saving method to another bean and add `@Transactional(propagation = Propagation.REQUIRES_NEW)`. The default propagation `REQUIRED` joins the transaction that is already open, so when the order rolls back the log rolls back with it. `REQUIRES_NEW` suspends the existing transaction and opens a new one, so only the log is committed on its own. The cost is one more connection. If the connection pool is small, the outer transaction can hold its connection while waiting for a new one and get stuck.

Take a pool of 10 connections with 10 order requests arriving at once.

```text
1. 10 order requests each take 1 connection for the outer transaction → 0 left in the pool
2. All 10 requests enter log saving (REQUIRES_NEW) and ask for 1 more connection
3. No request is going to return a connection → all 10 wait
4. After connectionTimeout (default 30 seconds), all 10 get SQLTransientConnectionException
```

So if any path uses `REQUIRES_NEW`, size the pool larger than "concurrent requests × connections one request holds at the same time".

**Q. What changes with `readOnly = true`?**

In JPA (Hibernate), the flush mode switches to `MANUAL`, so dirty checking and flushing are skipped at commit. This removes the cost of comparing entity snapshots in methods that only read.

For example, take a method that loads 1,000 members to show on a screen. With the default settings, at commit it compares the fields of all 1,000 entities with their snapshots one by one. With `readOnly = true`, this comparison is skipped. Even if you change an entity field by mistake, no UPDATE is sent.

In a setup with a read-only replica DB, this value is used to pick the DB. Extend `AbstractRoutingDataSource` and return the replica key for read-only transactions.

```java
public class RoutingDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        return TransactionSynchronizationManager.isCurrentTransactionReadOnly()
                ? "replica"  // @Transactional(readOnly = true)
                : "primary"; // everything else
    }
}
```

> `JpaTransactionManager` gets the connection first when the transaction starts, and the read-only flag is set after that. So if you register only the class above, every query goes to `primary`. Wrap it in `LazyConnectionDataSourceProxy` so the connection is fetched when the first SQL runs, and then queries go to `replica`.
{: .prompt-warning }

## 3. What is the JPA N+1 problem and how do you fix it?

### 3-1. First answer

If you load 10 `Team` rows and then read each team's `members`, you get 1 query for the team list plus 1 member query per team, 11 in total. One for the list plus N for the result rows, hence the name N+1.

```text
SELECT * FROM team;                       -- 1
SELECT * FROM member WHERE team_id = 1;   -- 1 per team
SELECT * FROM member WHERE team_id = 2;
...
SELECT * FROM member WHERE team_id = 10;  -- 11 in total
```

Switching to `FetchType.EAGER` does not fix it. JPQL first queries only `team`, then sees the eager setting and sends a member query per team. Only the timing of the queries moves earlier.

There are two fixes.

- Fetch join: `select t from Team t join fetch t.members` loads everything in one JOIN query.
- Batch size: with `spring.jpa.properties.hibernate.default_batch_fetch_size=100`, members are loaded together as `WHERE team_id IN (1, 2, ..., 10)`. The queries drop to 1 + 1.

### 3-2. Follow-up questions

**Q. What happens if you add paging to a collection fetch join?**

If one `Team` has 3 members, the JOIN result has 3 rows. A `LIMIT` in the DB would cut by rows, not by teams, so Hibernate does not add `LIMIT` to the SQL. Instead it loads the whole result into memory, cuts it in the application, and logs a warning. If there are 100,000 rows, all 100,000 go into memory.

In this case, page over `Team` alone and load `members` in one `IN` query with `default_batch_fetch_size`.

**Q. What if you fetch join two collections at once?**

Fetch joining two `List` collections together throws `MultipleBagFetchException`. Joining two collections multiplies the rows (count of A × count of B), and an unordered `List` (a bag) cannot tell which collection a duplicate row belongs to. Fetch join only one collection and load the rest with the batch size.

If a team has 2 members (A, B) and 3 projects (X, Y, Z), the JOIN result has 6 rows.

```text
team_id | member | project
1       | A      | X
1       | A      | Y
1       | A      | Z
1       | B      | X
1       | B      | Y
1       | B      | Z
```

A appears 3 times in the result. A bag is a collection that may hold the same element more than once. So Hibernate cannot tell whether A shows up 3 times because of the JOIN, or because `members` really contains A three times. Since it cannot make this call, it throws the exception before running the query.

**Q. How is `@EntityGraph` different from fetch join?**

`@EntityGraph(attributePaths = "members")` goes on a Spring Data JPA method and loads the associated entities without writing JPQL. The generated SQL is a LEFT OUTER JOIN. A plain `join fetch` is an INNER JOIN, so teams with no members drop out of the result.

## 4. Explain the persistence context and dirty checking

### 4-1. First answer

The persistence context keeps the entities loaded during one transaction, keyed by ID. It does three things.

- First-level cache: calling `findById(1L)` twice in the same transaction sends only one SELECT. Both results are the same object (`==` is `true`).
- Write-behind: INSERTs from `persist()` are not sent right away. They are collected and sent at flush.
- Dirty checking: when an entity is first loaded, a snapshot is saved. At flush, the snapshot is compared with the current fields, and an UPDATE is built for each changed entity. That is why, as in the example below, calling only `member.changeName(name)` without `save()` still sends an UPDATE.

```java
@Transactional
public void rename(Long id, String name) {
    Member member = memberRepository.findById(id).orElseThrow();
    member.changeName(name); // UPDATE at commit, no save() call
}
```

### 4-2. Follow-up questions

**Q. When does a flush happen?**

In three cases.

- Right before the transaction commits
- Right before a JPQL query runs (so the collected INSERTs show up in the JPQL result)
- When you call `flush()` yourself

The code below shows why it flushes before JPQL.

```java
@Transactional
public long join(Member member) {
    em.persist(member); // the INSERT is collected, not sent yet
    // without a flush here, the DB has no new member and count comes out 1 short
    return em.createQuery("select count(m) from Member m", Long.class)
             .getSingleResult();
}
```

JPQL runs as SQL against the DB, not against the first-level cache. So right before sending JPQL, JPA sends the collected INSERTs first so that the new member is in the result.

A flush sends SQL to the DB. It is not a commit. If you roll back after a flush, the sent changes are undone too.

**Q. What happens if you lazy-load outside a transaction?**

Reading a field of a proxy object after the persistence context is closed throws `LazyInitializationException`.

```java
// Service: the transaction is open only inside this method
@Transactional(readOnly = true)
public Member find(Long id) {
    return memberRepository.findById(id).orElseThrow();
    // team is @ManyToOne(fetch = FetchType.LAZY), so it is a proxy not yet read from the DB
}

// Controller: the transaction has already ended (with OSIV off)
Member member = memberService.find(1L);
member.getTeam().getName(); // cannot send the query to read the team, so LazyInitializationException
```

OSIV is what prevents this. In Spring Boot, `spring.jpa.open-in-view` defaults to `true`, so lazy loading works in controllers and views too. If you do not set the value yourself, one warning line is logged at startup.

**Q. What is the problem with leaving OSIV on?**

With OSIV on, the DB connection is not returned until the request finishes and the response is sent. If a controller waits 3 seconds for an external API, one connection is held for those 3 seconds. The HikariCP default pool size is 10, so if 10 such requests arrive at once, other requests cannot get a connection and wait.

High-traffic API servers turn it off with `spring.jpa.open-in-view=false` and load the needed associations inside the service-layer transaction, using fetch join or DTO conversion.

## 5. How does one request reach the controller?

### 5-1. First answer

Spring Boot runs on embedded Tomcat. One thread from Tomcat's thread pool handles one request until the response is sent.

```text
Client
   │
   ▼
Tomcat connector ── takes 1 thread from the thread pool
   │
   ▼
Filter chain       (servlet container: encoding, auth, CORS)
   │
   ▼
DispatcherServlet  (Spring MVC starts here)
   │
   ▼
HandlerMapping ── finds the controller method for the URL
   │
   ▼
Interceptor.preHandle
   │
   ▼
Controller ──> Service ──> Repository ──> DB
   │
   ▼
HttpMessageConverter ── converts the returned object to JSON
   │
   ▼
Interceptor.postHandle / afterCompletion
   │
   ▼
Response
```

### 5-2. Follow-up questions

**Q. How do Filter and Interceptor differ?**

| | Filter | Interceptor |
| --- | --- | --- |
| Belongs to | Servlet spec | Spring MVC |
| Runs | Before DispatcherServlet | After DispatcherServlet, around the controller |
| Receives | Request, response | Request, response, handler (controller method) |
| Exceptions | Not caught by `@ControllerAdvice` | Caught by `@ControllerAdvice` |

Work that must be filtered before entering Spring MVC, such as encoding, CORS and Spring Security authentication, goes in a Filter. Work such as checking login only for certain URL patterns, or work that needs to know which controller method will run, goes in an Interceptor.

**Q. How many concurrent requests can it handle?**

The embedded Tomcat thread limit is set by `server.tomcat.threads.max`, default 200. When all 200 are busy, new connections wait in the queue (`server.tomcat.accept-count`, default 100). When the queue is also full, connections are refused.

Adding threads alone does not raise throughput. If every request uses the DB and HikariCP has the default 10 connections, 190 of the 200 threads wait for a connection. A thread that cannot get a connection within `connectionTimeout` (default 30 seconds) gets `SQLTransientConnectionException`.

**Q. Where are exceptions thrown from a controller handled?**

Put an `@ExceptionHandler(BusinessException.class)` method in a `@RestControllerAdvice` class, and exceptions from many controllers are turned into an HTTP status code and response body in one place. Because this happens inside DispatcherServlet, exceptions from a Filter do not reach it. Authentication failures in Spring Security filters are handled separately in `AuthenticationEntryPoint`.

## Wrap-up

- Constructor injection fixes dependencies as `final` and lets tests create the object with `new`. With several beans of the same type, choose with `@Primary` or `@Qualifier`.
- `@Transactional` is applied by a proxy, so calls inside the same class do not open a transaction. Checked exceptions do not roll back by default.
- Reduce N+1 with fetch join or `default_batch_fetch_size`. Adding paging to a collection fetch join loads the whole result into memory.
- Dirty checking compares snapshots with current fields and builds UPDATEs at flush. With OSIV on, the DB connection is held until the response.
- One Tomcat thread handles a request through Filter, DispatcherServlet, Interceptor and Controller in order. Concurrent throughput is capped by the smaller of the thread count (default 200) and the connection pool size (default 10).

When preparing for an interview, build each answer in this order.

1. Define it in one sentence.
2. Say what happens inside and in what order.
3. Give one pitfall that comes from that structure, with example code.
4. Say how you avoided (or would avoid) that pitfall in practice.

## References

- [Spring Framework: Dependency Injection](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html)
- [Spring Framework: Bean Scopes](https://docs.spring.io/spring-framework/reference/core/beans/factory-scopes.html)
- [Spring Framework: Declarative Transaction Management](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative.html)
- [Spring Framework: Transaction Propagation](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html)
- [Spring Framework: LazyConnectionDataSourceProxy](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/LazyConnectionDataSourceProxy.html)
- [Spring Boot 2.6 Release Notes](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.6-Release-Notes)
- [Spring Boot: Common Application Properties](https://docs.spring.io/spring-boot/appendix/application-properties/index.html)
- [Hibernate ORM User Guide: Fetching](https://docs.jboss.org/hibernate/orm/6.6/userguide/html_single/Hibernate_User_Guide.html#fetching)
- [HikariCP: Configuration](https://github.com/brettwooldridge/HikariCP#gear-configuration-knobs-baby)
- [Spring Framework: DispatcherServlet](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-servlet.html)
