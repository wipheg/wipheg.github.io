---
title: "Spring 백엔드 면접 질문 5개와 꼬리 질문"
date: 2026-11-13 00:30:00 +0900
categories: [Backend, Spring]
tags: [spring, spring-boot, jpa, transactional, interview]
description: Spring 백엔드 면접에서 자주 나오는 질문 5개를 고르고, 첫 답변 뒤에 이어지는 꼬리 질문과 답변 포인트를 정리했다.
toc: true
lang: ko-KR
ref: spring-interview-questions
---

Spring 면접은 첫 질문보다 꼬리 질문에서 갈린다. "`@Transactional`은 어떻게 동작하나요?"에 "프록시로 동작합니다"라고 답하면, 면접관은 "그럼 같은 클래스 안에서 호출하면요?"라고 이어서 묻는다. 이 글은 자주 나오는 질문 5개마다 첫 답변과 꼬리 질문 3~4개를 함께 정리했다.

> **요약**
> - 생성자 주입은 의존성을 `final`로 고정하고, 순환 참조를 애플리케이션 시작 시점에 실패로 드러낸다.
> - `@Transactional`은 프록시가 메서드 호출을 가로채서 트랜잭션을 연다. 그래서 같은 클래스 안의 호출에는 적용되지 않는다.
> - N+1은 연관 엔티티를 지연 로딩하면서 목록 1번 + 행마다 1번씩 쿼리가 나가는 문제다. fetch join이나 배치 크기 설정으로 줄인다.
> - 영속성 컨텍스트는 트랜잭션 안에서 엔티티를 1차 캐시에 담고, 커밋 직전에 바뀐 필드를 찾아 UPDATE를 만든다.
> - 요청 하나는 Tomcat 스레드 하나가 Filter, DispatcherServlet, Interceptor, Controller 순으로 처리한다. 스레드 수와 DB 커넥션 수의 상한이 처리량을 정한다.
{: .prompt-tip }

## 먼저 알아둘 단어

| 단어 | 뜻 |
| --- | --- |
| 빈(Bean) | 스프링 컨테이너가 만들고 관리하는 객체 |
| 프록시 | 진짜 객체 앞에서 호출을 대신 받아 앞뒤에 작업을 끼워 넣는 객체 |
| 영속성 컨텍스트 | JPA가 엔티티를 보관하고 변경을 추적하는 공간 |
| 지연 로딩 | 연관 엔티티를 실제로 사용할 때 쿼리를 날려 가져오는 방식 |
| OSIV | Open Session In View. 영속성 컨텍스트를 HTTP 응답까지 열어 두는 설정 |

## 1. 의존성 주입은 왜 생성자 주입을 쓰나요?

### 1-1. 첫 답변

스프링이 지원하는 주입 방식은 필드 주입, 수정자(setter) 주입, 생성자 주입 세 가지다. 생성자 주입을 쓰는 이유는 세 가지로 답한다.

- 의존성을 `final` 필드로 선언할 수 있다. 객체가 만들어진 뒤 다른 구현체로 바뀌지 않는다.
- 테스트에서 스프링 없이 `new OrderService(fakeRepository)`로 객체를 만들 수 있다. 의존성을 빠뜨리면 생성자 인자가 맞지 않아 컴파일 오류가 난다.
- 의존성이 모두 생성자 인자에 드러난다. 인자가 7~8개로 늘어나면 클래스 하나가 너무 많은 일을 맡고 있다는 신호로 보고 클래스를 나눈다.

필드 주입은 두 번째 항목에서 차이가 난다. 스프링 없이 `new`로 만들면 컴파일은 되지만 필드가 `null`로 남는다.

```java
// 필드 주입 클래스를 테스트에서 직접 만든 경우
OrderService service = new OrderService(); // 컴파일 오류 없음
service.order(); // orderRepository가 null이라 여기서 NullPointerException
```

```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;

    // 생성자가 하나뿐이면 @Autowired를 생략해도 된다
    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
}
```

### 1-2. 꼬리 질문

**Q. A가 B를, B가 A를 생성자로 주입받으면 어떻게 되나요?**

애플리케이션이 시작할 때 `BeanCurrentlyInCreationException`으로 실패한다. A를 만들려면 B가 필요하고, B를 만들려면 아직 만들어지지 않은 A가 필요하기 때문이다. Spring Boot 2.6부터는 필드 주입이나 수정자 주입이어도 순환 참조를 기본으로 막는다. `spring.main.allow-circular-references=true`로 풀 수 있지만, 보통은 공통 로직을 제3의 클래스로 빼서 순환을 끊는다.

예를 들어 `OrderService`는 주문할 때 회원 등급을 확인하려고 `MemberService`를 주입받고, `MemberService`는 회원 탈퇴 전에 진행 중인 주문을 확인하려고 `OrderService`를 주입받는다고 하자. 두 서비스가 실제로 쓰는 것이 "회원의 주문 조회" 하나라면, 그 기능을 `MemberOrderQuery` 클래스로 옮긴다. 두 서비스가 모두 `MemberOrderQuery`만 주입받으면 서로를 가리키지 않는다.

```text
변경 전: OrderService ──▶ MemberService ──▶ OrderService   (순환)
변경 후: OrderService ──▶ MemberOrderQuery ◀── MemberService
```

**Q. 같은 타입의 빈이 두 개면 어떤 빈이 주입되나요?**

아무 설정도 없으면 `NoUniqueBeanDefinitionException`이 난다. 해결 방법은 세 가지다.

- `@Primary`: 기본으로 쓸 빈 하나에 붙인다.
- `@Qualifier("이름")`: 주입받는 쪽에서 빈 이름을 지정한다. `@Primary`보다 우선한다.
- `List<PaymentClient>`나 `Map<String, PaymentClient>`로 받아서 전부 주입받는다. 결제 수단별 구현체를 고르는 전략 패턴에 쓴다.

**Q. 빈은 기본으로 싱글톤인데, 그래서 생기는 문제가 있나요?**

기본 스코프가 싱글톤이라서 빈 하나를 여러 요청 스레드가 동시에 쓴다. 빈 안에 요청마다 바뀌는 값을 인스턴스 필드로 두면, 사용자 A의 요청이 넣은 값을 사용자 B의 요청이 읽을 수 있다. 요청마다 달라지는 값은 지역 변수나 메서드 인자로 넘긴다.

```java
@Service
public class PriceService {
    private int discount; // 모든 요청 스레드가 이 필드 하나를 함께 쓴다

    // 잘못된 예: 필드에 넣었다가 다시 읽는다
    public int calcWrong(int price, int discount) {
        this.discount = discount;
        return price - this.discount;
    }

    // 올바른 예: 인자는 스레드마다 따로 있다
    public int calc(int price, int discount) {
        return price - discount;
    }
}
```

`calcWrong()`에 두 요청이 거의 동시에 들어오면 아래 순서가 생길 수 있다.

```text
1. 요청 A: this.discount = 1000
2. 요청 B: this.discount = 3000
3. 요청 A: price - this.discount 계산 → 1000이 아니라 3000을 뺀다
```

## 2. `@Transactional`은 어떻게 동작하나요?

### 2-1. 첫 답변

스프링은 `@Transactional`이 붙은 빈 대신 프록시 객체를 컨테이너에 등록한다. 다른 빈이 이 서비스를 호출하면 프록시가 먼저 호출을 받는다. 프록시는 트랜잭션을 시작하고, 진짜 메서드를 호출한 다음, 정상 종료면 커밋하고 예외면 롤백한다.

```text
Controller ──호출──> [프록시] ──> 트랜잭션 시작
                        │
                        ├──> OrderService.order() 실행 (진짜 객체)
                        │
                        └──> 정상 종료: 커밋 / RuntimeException: 롤백
```

### 2-2. 꼬리 질문

**Q. 같은 클래스 안에서 `@Transactional` 메서드를 호출하면요?**

트랜잭션이 적용되지 않는다. `this.save()`는 프록시를 거치지 않고 진짜 객체의 메서드를 바로 호출하기 때문이다. 이를 self-invocation 문제라고 한다.

```java
@Service
public class OrderService {
    public void order() {
        save(); // this.save(): 프록시를 거치지 않는다
    }

    @Transactional
    public void save() { ... } // 트랜잭션이 열리지 않는다
}
```

해결 방법은 `save()`를 다른 빈으로 옮기는 것이다. 호출하는 쪽 메서드 `order()`에 `@Transactional`을 붙여도 된다.

**Q. checked 예외가 나면 롤백되나요?**

기본 설정에서는 롤백되지 않고 커밋된다. 스프링은 `RuntimeException`과 `Error`만 롤백 대상으로 본다. `IOException` 같은 checked 예외에도 롤백하려면 `@Transactional(rollbackFor = Exception.class)`처럼 지정한다.

> 메서드 안에서 예외를 `try-catch`로 잡고 다시 던지지 않으면 프록시는 예외를 보지 못한다. 그러면 정상 종료로 판단해 커밋한다.
{: .prompt-warning }

**Q. 주문은 실패해도 실패 로그는 남겨야 하면 어떻게 하나요?**

로그를 저장하는 메서드를 다른 빈으로 빼고 `@Transactional(propagation = Propagation.REQUIRES_NEW)`를 붙인다. 기본 전파 속성 `REQUIRED`는 이미 열린 트랜잭션에 참여하기 때문에 주문이 롤백되면 로그도 같이 롤백된다. `REQUIRES_NEW`는 기존 트랜잭션을 잠시 멈추고 새 트랜잭션을 열기 때문에 로그만 따로 커밋된다. 대신 커넥션을 하나 더 쓴다. 커넥션 풀이 작으면 바깥 트랜잭션이 커넥션을 쥔 채 새 커넥션을 기다리다 막힐 수 있다.

커넥션 풀이 10개이고 주문 요청 10개가 동시에 들어온 경우를 보자.

```text
1. 주문 요청 10개가 바깥 트랜잭션용 커넥션을 1개씩 가져간다 → 풀에 남은 커넥션 0개
2. 10개 요청이 모두 로그 저장(REQUIRES_NEW)에 들어가서 커넥션을 1개 더 달라고 한다
3. 커넥션을 돌려줄 요청이 없다 → 10개 모두 대기
4. connectionTimeout(기본 30초)이 지나면 10개 모두 SQLTransientConnectionException
```

그래서 `REQUIRES_NEW`를 쓰는 경로가 있으면 풀 크기를 "동시 요청 수 × 요청 하나가 동시에 쥐는 커넥션 수"보다 크게 잡는다.

**Q. `readOnly = true`는 무엇이 달라지나요?**

JPA(Hibernate)에서는 플러시 모드를 `MANUAL`로 바꿔서 커밋할 때 변경 감지와 플러시를 하지 않는다. 그래서 조회만 하는 메서드에서 엔티티 스냅샷 비교 비용이 줄어든다.

예를 들어 회원 1,000명을 조회해서 화면에 보여주는 메서드가 있다고 하자. 기본 설정이면 커밋할 때 엔티티 1,000개의 필드를 하나씩 스냅샷과 비교한다. `readOnly = true`면 이 비교를 건너뛴다. 실수로 엔티티 필드를 바꿔도 UPDATE가 나가지 않는다.

읽기 전용 복제 DB를 두는 구성에서는 이 값을 보고 DB를 고른다. `AbstractRoutingDataSource`를 상속해서 읽기 전용 트랜잭션이면 복제 DB 키를 돌려준다.

```java
public class RoutingDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        return TransactionSynchronizationManager.isCurrentTransactionReadOnly()
                ? "replica"  // @Transactional(readOnly = true)
                : "primary"; // 그 외
    }
}
```

> `JpaTransactionManager`는 트랜잭션을 시작할 때 커넥션을 먼저 가져오고, 읽기 전용 표시는 그 뒤에 설정된다. 그래서 위 클래스만 등록하면 항상 `primary`로 간다. `LazyConnectionDataSourceProxy`로 감싸서 첫 SQL을 실행할 때 커넥션을 가져오게 해야 `replica`로 간다.
{: .prompt-warning }

## 3. JPA N+1 문제는 무엇이고 어떻게 해결하나요?

### 3-1. 첫 답변

`Team` 10개를 조회한 뒤 각 팀의 `members`를 꺼내면, 팀 목록 쿼리 1번에 팀마다 회원 쿼리가 1번씩 더 나가서 총 11번이 실행된다. 목록 1번 + 결과 행 N번이라서 N+1 문제라고 부른다.

```text
SELECT * FROM team;                       -- 1번
SELECT * FROM member WHERE team_id = 1;   -- 팀마다 1번씩
SELECT * FROM member WHERE team_id = 2;
...
SELECT * FROM member WHERE team_id = 10;  -- 총 11번
```

`FetchType.EAGER`로 바꿔도 해결되지 않는다. JPQL은 먼저 `team`만 조회하고, 그 뒤에 즉시 로딩 설정을 보고 회원 쿼리를 팀마다 날린다. 쿼리가 나가는 시점만 앞당겨진다.

해결 방법은 두 가지다.

- fetch join: `select t from Team t join fetch t.members`로 한 번의 JOIN 쿼리로 가져온다.
- 배치 크기: `spring.jpa.properties.hibernate.default_batch_fetch_size=100`을 설정하면 회원을 `WHERE team_id IN (1, 2, ..., 10)`으로 묶어서 가져온다. 쿼리가 1 + 1번으로 줄어든다.

### 3-2. 꼬리 질문

**Q. 컬렉션 fetch join에 페이징을 붙이면 어떻게 되나요?**

`Team` 1개에 회원이 3명이면 JOIN 결과는 3행이 된다. DB에서 `LIMIT`을 걸면 팀 단위가 아니라 행 단위로 잘리기 때문에, Hibernate는 SQL에 `LIMIT`을 붙이지 않는다. 대신 전체 결과를 메모리로 가져온 뒤 애플리케이션에서 자르고 경고 로그를 남긴다. 데이터가 10만 건이면 10만 건을 모두 메모리에 올린다.

이 경우에는 `Team`만 페이징으로 조회하고, `members`는 `default_batch_fetch_size`로 `IN` 쿼리 한 번에 가져온다.

**Q. 컬렉션 두 개를 동시에 fetch join하면요?**

`List` 타입 컬렉션 두 개를 함께 fetch join하면 `MultipleBagFetchException`이 난다. 두 컬렉션을 JOIN하면 행이 (A 개수 × B 개수)만큼 곱해지고, 순서 없는 `List`(bag)에서는 중복 행을 어느 쪽 원소로 볼지 구분할 수 없기 때문이다. 컬렉션 하나만 fetch join하고 나머지는 배치 크기로 가져온다.

팀 하나에 회원 2명(A, B)과 프로젝트 3개(X, Y, Z)가 있으면 JOIN 결과는 6행이다.

```text
team_id | member | project
1       | A      | X
1       | A      | Y
1       | A      | Z
1       | B      | X
1       | B      | Y
1       | B      | Z
```

결과에 A가 3번 나온다. bag은 같은 원소가 여러 번 들어 있어도 되는 컬렉션이다. 그래서 Hibernate는 A가 JOIN 때문에 3번 찍힌 것인지, 실제로 `members`에 A가 3번 들어 있는 것인지 구분할 수 없다. 이 판단을 할 수 없으니 쿼리를 실행하기 전에 예외를 던진다.

**Q. `@EntityGraph`는 fetch join과 무엇이 다른가요?**

`@EntityGraph(attributePaths = "members")`는 Spring Data JPA 메서드에 붙여서 JPQL 없이 연관 엔티티를 같이 가져온다. 생성되는 SQL은 LEFT OUTER JOIN이다. 기본 `join fetch`는 INNER JOIN이라서 회원이 없는 팀은 결과에서 빠진다.

## 4. 영속성 컨텍스트와 변경 감지(Dirty Checking)를 설명해 주세요

### 4-1. 첫 답변

영속성 컨텍스트는 트랜잭션 하나 동안 조회한 엔티티를 ID별로 보관한다. 하는 일은 세 가지다.

- 1차 캐시: 같은 트랜잭션에서 `findById(1L)`를 두 번 호출하면 SELECT는 한 번만 나간다. 두 결과는 같은 객체(`==`가 `true`)다.
- 쓰기 지연: `persist()`한 INSERT를 바로 보내지 않고 모아 두었다가 플러시할 때 보낸다.
- 변경 감지: 엔티티를 처음 읽을 때 스냅샷을 저장한다. 플러시할 때 스냅샷과 현재 필드를 비교해서 바뀐 엔티티의 UPDATE를 만든다. 그래서 아래 예시처럼 `member.changeName(name)`만 하고 `save()`를 부르지 않아도 UPDATE가 나간다.

```java
@Transactional
public void rename(Long id, String name) {
    Member member = memberRepository.findById(id).orElseThrow();
    member.changeName(name); // save() 호출 없이 커밋 시점에 UPDATE
}
```

### 4-2. 꼬리 질문

**Q. 플러시는 언제 일어나나요?**

세 가지 경우다.

- 트랜잭션을 커밋하기 직전
- JPQL 쿼리를 실행하기 직전 (쓰기 지연 저장소의 INSERT가 JPQL 결과에 보이도록)
- `flush()`를 직접 호출할 때

JPQL 실행 전에 플러시하는 이유는 아래 코드로 보면 된다.

```java
@Transactional
public long join(Member member) {
    em.persist(member); // INSERT를 아직 보내지 않고 모아 둔다
    // 여기서 플러시하지 않으면 DB에는 새 회원이 없어서 count가 1 적게 나온다
    return em.createQuery("select count(m) from Member m", Long.class)
             .getSingleResult();
}
```

JPQL은 1차 캐시가 아니라 DB에 SQL로 실행된다. 그래서 JPA는 JPQL을 보내기 직전에 모아 둔 INSERT를 먼저 보내서 결과에 새 회원이 들어가게 한다.

플러시는 SQL을 DB로 보내는 것이고 커밋이 아니다. 플러시 뒤에 롤백하면 보낸 변경도 취소된다.

**Q. 트랜잭션 밖에서 지연 로딩을 하면 어떻게 되나요?**

영속성 컨텍스트가 닫힌 뒤에 프록시 객체의 필드를 읽으면 `LazyInitializationException`이 난다.

```java
// Service: 트랜잭션이 이 메서드 안에서만 열려 있다
@Transactional(readOnly = true)
public Member find(Long id) {
    return memberRepository.findById(id).orElseThrow();
    // team이 @ManyToOne(fetch = FetchType.LAZY)라서 아직 DB에서 읽지 않은 프록시 객체다
}

// Controller: 트랜잭션이 이미 끝났다 (OSIV가 꺼져 있을 때)
Member member = memberService.find(1L);
member.getTeam().getName(); // 팀을 읽을 쿼리를 보낼 수 없어서 LazyInitializationException
```

이를 막는 방법이 OSIV다. Spring Boot에서는 `spring.jpa.open-in-view`가 기본 `true`이고, 그래서 컨트롤러와 뷰에서도 지연 로딩이 된다. 기본값을 따로 정하지 않으면 시작할 때 경고 로그가 한 줄 남는다.

**Q. OSIV를 켜 두면 무엇이 문제인가요?**

OSIV가 켜져 있으면 요청이 끝나서 응답을 보낼 때까지 DB 커넥션을 돌려주지 않는다. 컨트롤러에서 외부 API를 3초 동안 기다리면 그 3초 동안 커넥션 하나가 묶인다. HikariCP 기본 풀 크기는 10개라서 이런 요청 10개가 동시에 들어오면 다른 요청은 커넥션을 받지 못하고 기다린다.

트래픽이 많은 API 서버는 `spring.jpa.open-in-view=false`로 끄고, 필요한 연관 엔티티를 서비스 계층의 트랜잭션 안에서 fetch join이나 DTO 변환으로 미리 가져온다.

## 5. 요청 하나가 컨트롤러까지 어떻게 들어오나요?

### 5-1. 첫 답변

Spring Boot는 내장 Tomcat에서 실행된다. 요청 하나는 Tomcat 스레드 풀의 스레드 하나가 응답을 보낼 때까지 처리한다.

```text
클라이언트
   │
   ▼
Tomcat 커넥터 ── 스레드 풀에서 스레드 1개 할당
   │
   ▼
Filter 체인        (서블릿 컨테이너 영역: 인코딩, 인증, CORS)
   │
   ▼
DispatcherServlet  (여기부터 스프링 MVC)
   │
   ▼
HandlerMapping ── 요청 URL에 맞는 컨트롤러 메서드 찾기
   │
   ▼
Interceptor.preHandle
   │
   ▼
Controller ──> Service ──> Repository ──> DB
   │
   ▼
HttpMessageConverter ── 반환 객체를 JSON으로 변환
   │
   ▼
Interceptor.postHandle / afterCompletion
   │
   ▼
응답
```

### 5-2. 꼬리 질문

**Q. Filter와 Interceptor는 무엇이 다른가요?**

| 구분 | Filter | Interceptor |
| --- | --- | --- |
| 소속 | 서블릿 스펙 | 스프링 MVC |
| 실행 위치 | DispatcherServlet 앞 | DispatcherServlet 뒤, 컨트롤러 앞뒤 |
| 받는 정보 | 요청, 응답 객체 | 요청, 응답, 실행될 핸들러(컨트롤러 메서드) |
| 예외 처리 | `@ControllerAdvice`가 잡지 못함 | `@ControllerAdvice`가 잡음 |

인코딩, CORS, Spring Security 인증처럼 스프링 MVC에 들어오기 전에 걸러야 하는 일은 Filter에 둔다. 특정 URL 패턴에만 로그인 여부를 확인하거나, 어떤 컨트롤러 메서드가 실행되는지 알아야 하는 일은 Interceptor에 둔다.

**Q. 동시 요청이 몇 개까지 처리되나요?**

내장 Tomcat 스레드 수 상한은 `server.tomcat.threads.max`로 정하고 기본값은 200이다. 200개가 모두 일하는 중이면 새 연결은 대기열(`server.tomcat.accept-count`, 기본 100)에서 기다린다. 대기열도 차면 연결이 거부된다.

스레드 수만 늘린다고 처리량이 늘지는 않는다. 요청마다 DB를 쓰는데 HikariCP 커넥션이 기본 10개라면, 스레드 200개 중 190개는 커넥션을 기다린다. 커넥션을 `connectionTimeout`(기본 30초) 안에 받지 못하면 `SQLTransientConnectionException`이 난다.

**Q. 컨트롤러에서 던진 예외는 어디서 처리하나요?**

`@RestControllerAdvice` 클래스에 `@ExceptionHandler(BusinessException.class)` 메서드를 두면 여러 컨트롤러의 예외를 한 곳에서 HTTP 상태 코드와 응답 본문으로 바꾼다. DispatcherServlet 안에서 처리되기 때문에 Filter에서 난 예외는 여기로 오지 않는다. Spring Security 필터에서 난 인증 실패는 `AuthenticationEntryPoint`에서 따로 처리한다.

## 정리

- 생성자 주입은 의존성을 `final`로 고정하고 테스트에서 `new`로 객체를 만들 수 있게 한다. 같은 타입 빈이 여러 개면 `@Primary`나 `@Qualifier`로 고른다.
- `@Transactional`은 프록시가 적용하므로 같은 클래스 안의 호출에는 트랜잭션이 열리지 않는다. checked 예외는 기본으로 롤백되지 않는다.
- N+1은 fetch join이나 `default_batch_fetch_size`로 줄인다. 컬렉션 fetch join에 페이징을 붙이면 전체 결과를 메모리에 올린다.
- 변경 감지는 스냅샷과 현재 필드를 비교해서 플러시 시점에 UPDATE를 만든다. OSIV를 켜 두면 응답까지 DB 커넥션을 쥐고 있다.
- 요청은 Tomcat 스레드 하나가 Filter, DispatcherServlet, Interceptor, Controller 순으로 처리한다. 동시 처리량은 스레드 수(기본 200)와 커넥션 풀 크기(기본 10) 중 작은 쪽에 막힌다.

면접 준비를 할 때는 질문마다 아래 순서로 답을 만들어 본다.

1. 한 문장으로 정의한다.
2. 내부에서 무엇이 어떤 순서로 일어나는지 말한다.
3. 그 구조 때문에 생기는 함정 하나를 예시 코드로 든다.
4. 실무에서 그 함정을 어떻게 피했는지(또는 피할지) 말한다.

## 참고 문서

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
