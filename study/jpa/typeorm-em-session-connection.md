# EntityManager, Session, Connection의 관계 (JPA vs TypeORM)

## 개요

JPA에서 EntityManager, Session, Connection, Thread는 모두 다른 계층의 개념이다.
"쓰레드 = 세션", "EntityManager = 커넥션"처럼 혼동하기 쉽지만 각각 역할과 수명이 다르다.

---

## JPA 계층 구조

```
EntityManagerFactory (앱 1개)
  └─ EntityManager (= Session) (요청마다 1개)
       ├─ 영속성 컨텍스트 (1차 캐시, 쓰기 지연, 스냅샷)
       └─ Transaction
            └─ Connection (DB 커넥션)  ← 커넥션 풀에서 빌려옴
```

### 각 개념 정리

| 개념 | 정체 | 수명 |
|---|---|---|
| **Thread** | OS/JVM 실행 단위 | 요청 시작 ~ 응답 완료 (서블릿 모델) |
| **Session** (Hibernate) | 영속성 컨텍스트의 구현체 | `openSession()` ~ `close()` |
| **EntityManager** (JPA) | Session의 JPA 표준 래퍼 | `createEntityManager()` ~ `close()` |
| **Transaction** | DB 트랜잭션 | `begin()` ~ `commit()/rollback()` |
| **Connection** | DB와의 물리적 TCP 연결 | 풀에서 빌림 ~ 반환 |

---

## Thread ≠ Session

Spring에서 `@Transactional`을 쓰면 **ThreadLocal**에 Session을 바인딩한다:

```java
@Transactional
public void doSomething() {
    // Spring이 내부적으로:
    // 1. 현재 쓰레드에 Session이 없으면 → 새로 생성하여 ThreadLocal에 저장
    // 2. em.find(), em.persist() → ThreadLocal에서 Session을 꺼내 사용
    // 3. 메서드 끝 → flush + commit + Session close
}
```

**쓰레드 1개 = Session 1개**로 동작하므로 같다고 느껴지지만, 실제로는:

- Session은 **객체**이고, Thread는 **실행 단위**
- Spring이 ThreadLocal로 **연결해주는 것**일 뿐
- ThreadLocal 없이 직접 쓰면 하나의 쓰레드에서 여러 Session을 만들 수 있다

```java
// Spring 없이 직접 사용하면 쓰레드와 Session은 독립적
Session session1 = sessionFactory.openSession();  // 세션 1
session1.close();

Session session2 = sessionFactory.openSession();  // 같은 쓰레드에서 세션 2
session2.close();
```

---

## EntityManager ≠ Connection

EntityManager는 커넥션이 아니다. 계층이 다르다.

| | EntityManager (Session) | Connection |
|---|---|---|
| **정체** | 영속성 컨텍스트를 관리하는 **논리적 작업 단위** | DB와의 **물리적 TCP 연결** |
| **위치** | 애플리케이션 레벨 | JDBC/드라이버 레벨 |
| **역할** | 엔티티 상태 추적, 1차 캐시, dirty checking | SQL 전송, 결과 수신 |
| **수명** | 요청 시작 ~ 끝 | 필요할 때 풀에서 빌리고 반환 |
| **개수** | 요청마다 1개 | 커넥션 풀에서 공유 |

### 커넥션은 필요할 때만 잠깐 빌린다

```java
@Transactional
public void doSomething() {
    User user = em.find(User.class, 1L);
    // → 커넥션 풀에서 빌림 → SELECT 실행 → 반환 (구현에 따라 다름)

    user.setName("Tom");
    // → 아무 일도 안 일어남 (메모리에서 변경만)
    // → 커넥션 사용 안 함

    // 메서드 끝 → flush
    // → 커넥션 풀에서 빌림 → UPDATE 실행 → commit → 반환
}
```

EntityManager는 **요청 내내 살아있지만**, 커넥션은 **SQL을 보낼 때만** 풀에서 빌려 쓴다.
(Hibernate의 기본 동작, `ConnectionReleaseMode`에 따라 다름)

### 비유

```
EntityManager = 사무실 책상 (내 작업 공간, 요청 동안 점유)
  ├─ 영속성 컨텍스트 = 책상 위의 서류 (작업 중인 엔티티들)
  └─ Connection = 공용 팩스기 (SQL 보낼 때만 잠깐 사용, 다 쓰면 반납)
```

---

## TypeORM에서의 대응

TypeORM에도 이 구분이 있지만, 영속성 컨텍스트가 없으므로 훨씬 단순하다.

### 구조

```
DataSource (앱 1개)
  ├─ EntityManager (앱 1개, 상태 없음)
  └─ 커넥션 풀
       └─ QueryRunner (쿼리 실행 시 풀에서 커넥션 1개 빌림)
```

### 동작

```typescript
await userRepo.findOne({ where: { id: 1 } })
// 내부: QueryRunner 생성 → 풀에서 커넥션 빌림 → SQL 실행 → 커넥션 반환 → QueryRunner 해제
```

### JPA와의 차이

JPA의 EntityManager가 "영속성 컨텍스트 + 커넥션 관리"를 하나로 묶는 반면,
TypeORM의 EntityManager는 영속성 컨텍스트 없이 **QueryRunner에 위임만 하는 파사드**이다.

---

## 비교 요약

| JPA | TypeORM | 비고 |
|---|---|---|
| `EntityManagerFactory` | `DataSource` | 앱 1개, 무거운 초기화 |
| `EntityManager` (Session) | `EntityManager` | JPA: 요청마다 생성, 상태 있음 / TypeORM: 앱 1개, 상태 없음 |
| 영속성 컨텍스트 | 해당 없음 | TypeORM에 없는 개념 |
| `Connection` (JDBC) | `QueryRunner` 내부의 DB 커넥션 | 둘 다 풀에서 빌려서 사용 |
| `ConnectionPool` (HikariCP 등) | `pg.Pool`, `mysql2.Pool` 등 | npm 패키지의 풀 |
| `ThreadLocal` → Session 바인딩 | 해당 없음 | Node.js는 싱글 쓰레드, 영속성 컨텍스트 없음 |

### 요청 격리 방식

```
JPA + Spring:
  쓰레드 1 → ThreadLocal → Session A (영속성 컨텍스트)
  쓰레드 2 → ThreadLocal → Session B (영속성 컨텍스트)
  → 쓰레드별 독립된 영속성 컨텍스트로 격리

TypeORM:
  싱글 쓰레드 → EntityManager 1개 (상태 없음)
  → 격리할 상태 자체가 없음
```