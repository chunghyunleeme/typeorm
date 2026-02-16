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

| 개념                    | 정체                     | 수명                                |
| ----------------------- | ------------------------ | ----------------------------------- |
| **Thread**              | OS/JVM 실행 단위         | 요청 시작 ~ 응답 완료 (서블릿 모델) |
| **Session** (Hibernate) | 영속성 컨텍스트의 구현체 | `openSession()` ~ `close()`         |
| **EntityManager** (JPA) | Session의 JPA 표준 래퍼  | `createEntityManager()` ~ `close()` |
| **Transaction**         | DB 트랜잭션              | `begin()` ~ `commit()/rollback()`   |
| **Connection**          | DB와의 물리적 TCP 연결   | 풀에서 빌림 ~ 반환                  |

---

## JPA 요청 처리 흐름 (시간순)

```
[앱 시작]
  │
  ▼
EntityManagerFactory 생성 (1회)
  ├─ persistence.xml 읽기
  ├─ 엔티티 메타데이터 빌드
  └─ 커넥션 풀 생성 (HikariCP 등)
  │
  │   앱 실행 중... EMF는 싱글턴으로 유지
  │
[요청 도착]
  │
  ▼
EMF.createEntityManager() → EntityManager 생성
  └─ 영속성 컨텍스트 생성 (1차 캐시, 쓰기 지연 저장소, 스냅샷)
  │
  ▼
Transaction 시작 ← 자동이 아님! (아래 참고)
  │
  ▼
em.find(User.class, 1L)
  ├─ 1차 캐시 확인 → 없으면:
  ├─ 커넥션 풀에서 Connection 빌림
  ├─ SELECT SQL 실행
  ├─ Connection 반환 (구현에 따라 다름)
  └─ 결과를 1차 캐시에 저장
  │
  ▼
em.persist(newUser)
  └─ 쓰기 지연 저장소에 INSERT 예약 (SQL 실행 안 함)
  │
  ▼
Transaction commit
  ├─ flush → 쓰기 지연 저장소의 SQL 일괄 실행
  │   ├─ 커넥션 풀에서 Connection 빌림
  │   ├─ INSERT 실행
  │   ├─ dirty checking → 변경된 엔티티 UPDATE 실행
  │   └─ Connection 반환
  ├─ DB commit
  └─ EntityManager close → 영속성 컨텍스트 소멸
```

**트랜잭션은 자동이 아니다.** 위 흐름에서 EM 생성, 트랜잭션 시작/커밋, EM 닫기는 모두 수동이다.

순수 JPA에서는 개발자가 직접 관리해야 한다:

```java
EntityManager em = emf.createEntityManager();
EntityTransaction tx = em.getTransaction();
tx.begin();          // 직접 시작
// ... 작업 ...
tx.commit();         // 직접 커밋
em.close();          // 직접 닫기
```

Spring의 `@Transactional`이 이 보일러플레이트를 자동화해주는 것이다:

```java
@Transactional   // Spring AOP가 begin, commit/rollback, close를 대신 처리
public void updateUser(Long id) {
    User user = em.find(User.class, id);
    user.setName("Tom");
    // 메서드 끝 → 자동 flush + commit + EM close
}
```

"요청 1개 = 트랜잭션 1개"는 JPA 스펙이 아니라 **Spring `@Transactional`의 편의 기능**이다.
하나의 요청 안에서 트랜잭션을 여러 개 만들 수도 있다 (`REQUIRES_NEW` 등).

### TypeORM에서 같은 흐름

```
[앱 시작]
  │
  ▼
new DataSource(options) + initialize()
  ├─ 커넥션 풀 생성 (pg.Pool 등)
  ├─ 엔티티 메타데이터 빌드
  └─ EntityManager 1개 생성 (상태 없음, 앱 전체 재사용)
  │
  │   앱 실행 중...
  │
[요청 도착]
  │
  ▼
repo.findOne({ where: { id: 1 } })
  ├─ QueryRunner 생성 → 커넥션 풀에서 Connection 빌림
  ├─ SELECT SQL 실행
  ├─ Connection 반환 → QueryRunner 해제
  └─ 새 엔티티 인스턴스 반환 (캐시 없음)
  │
  ▼
repo.save(newUser)
  ├─ QueryRunner 생성 → 커넥션 풀에서 Connection 빌림
  ├─ INSERT SQL 즉시 실행
  ├─ commit
  └─ Connection 반환 → QueryRunner 해제
```

핵심 차이: JPA는 **요청 단위로 EM을 생성하여 상태를 관리**하고, TypeORM은 **쿼리 단위로 커넥션을 빌려서 즉시 실행**한다.

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

|          | EntityManager (Session)                         | Connection                   |
| -------- | ----------------------------------------------- | ---------------------------- |
| **정체** | 영속성 컨텍스트를 관리하는 **논리적 작업 단위** | DB와의 **물리적 TCP 연결**   |
| **위치** | 애플리케이션 레벨                               | JDBC/드라이버 레벨           |
| **역할** | 엔티티 상태 추적, 1차 캐시, dirty checking      | SQL 전송, 결과 수신          |
| **수명** | 요청 시작 ~ 끝                                  | 필요할 때 풀에서 빌리고 반환 |
| **개수** | 요청마다 1개                                    | 커넥션 풀에서 공유           |

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

| JPA                            | TypeORM                        | 비고                                                       |
| ------------------------------ | ------------------------------ | ---------------------------------------------------------- |
| `EntityManagerFactory`         | `DataSource`                   | 앱 1개, 무거운 초기화                                      |
| `EntityManager` (Session)      | `EntityManager`                | JPA: 요청마다 생성, 상태 있음 / TypeORM: 앱 1개, 상태 없음 |
| 영속성 컨텍스트                | 해당 없음                      | TypeORM에 없는 개념                                        |
| `Connection` (JDBC)            | `QueryRunner` 내부의 DB 커넥션 | 둘 다 풀에서 빌려서 사용                                   |
| `ConnectionPool` (HikariCP 등) | `pg.Pool`, `mysql2.Pool` 등    | npm 패키지의 풀                                            |
| `ThreadLocal` → Session 바인딩 | 해당 없음                      | Node.js는 싱글 쓰레드, 영속성 컨텍스트 없음                |

### TypeORM의 EntityManagerFactory는 이름만 같다

TypeORM에도 `EntityManagerFactory` 클래스가 있지만 (`src/entity-manager/EntityManagerFactory.ts`),
JPA의 것과는 완전히 다르다:

```typescript
export class EntityManagerFactory {
    create(connection: DataSource, queryRunner?: QueryRunner): EntityManager {
        if (type === "mongodb") return new MongoEntityManager(connection)
        if (type === "sqljs")
            return new SqljsEntityManager(connection, queryRunner)
        return new EntityManager(connection, queryRunner)
    }
}
```

DB 타입에 따라 알맞은 EntityManager 클래스를 `new`로 만들어주는 **if-else 분기를 클래스로 뺀 것**에 불과하다.
메타데이터, 커넥션 풀 등 상태를 보관하지 않으며, 매번 `new EntityManagerFactory()`로 만들어 쓰고 버린다.

|              | JPA `EntityManagerFactory`               | TypeORM `EntityManagerFactory`                     |
| ------------ | ---------------------------------------- | -------------------------------------------------- |
| **역할**     | 메타데이터 빌드, 커넥션 풀 관리, EM 생성 | DB 타입 보고 알맞은 EM 클래스 `new` 해주기         |
| **상태**     | 있음 (메타데이터, 커넥션 풀, 2차 캐시)   | 없음 (메서드 1개짜리 유틸리티)                     |
| **수명**     | 앱 전체 (싱글턴)                         | 인스턴스를 유지하지도 않음                         |
| **JPA 대응** | —                                        | JPA에 대응하는 개념 없음 (`DataSource`가 EMF 역할) |

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

---

## QueryRunner와 트랜잭션

### 트랜잭션은 커넥션 단위

DB 트랜잭션은 하나의 커넥션 위에서 돌아야 한다. `START TRANSACTION`과 `COMMIT`이 같은 TCP 연결에서 실행되어야 한다.
TypeORM에서 이를 보장하는 것이 **QueryRunner**다.

### QueryRunner의 구조

```
QueryRunner
  ├─ databaseConnection  ← 커넥션 풀에서 빌린 커넥션 1개 (고정)
  ├─ isTransactionActive ← 트랜잭션 진행 중인지
  ├─ transactionDepth    ← 중첩 트랜잭션 깊이
  │
  ├─ connect()           → 커넥션 빌리기 (최초 1회, 이후 재사용)
  ├─ query(sql)          → this.databaseConnection으로 SQL 실행
  ├─ startTransaction()  → "START TRANSACTION" 실행
  ├─ commitTransaction() → "COMMIT" 실행
  ├─ rollbackTransaction()→ "ROLLBACK" 실행
  └─ release()           → 커넥션을 풀에 반환
```

### 같은 커넥션을 재사용하는 코드 근거

`connect()` — 최초에만 빌리고 이후 재사용 (`src/driver/postgres/PostgresQueryRunner.ts:82`):

```typescript
connect() {
    if (this.databaseConnection)       // 이미 빌렸으면 같은 커넥션 재사용
        return this.databaseConnection

    // 최초에만 풀에서 빌림
    this.databaseConnectionPromise = this.driver.obtainMasterConnection()
        .then(([connection, release]) => {
            this.databaseConnection = connection  // 저장해두고 계속 사용
            this.releaseCallback = release         // 반납 함수 저장
        })
}
```

`startTransaction()` / `commitTransaction()` (`src/driver/postgres/PostgresQueryRunner.ts:170-213`):

```typescript
async startTransaction() {
    await this.query("START TRANSACTION")  // this.query → 같은 커넥션
}
async commitTransaction() {
    await this.query("COMMIT")             // this.query → 같은 커넥션
}
```

모두 `this.query()`를 거치고, `this.query()`는 항상 `this.databaseConnection`(같은 커넥션)을 사용한다.

### 트랜잭션 실행 흐름

`dataSource.transaction()` 내부 (`src/entity-manager/EntityManager.ts:154-160`):

```typescript
async transaction(runInTransaction) {
    const queryRunner = this.connection.createQueryRunner()  // 커넥션 1개 확보

    await queryRunner.startTransaction()                         // START TRANSACTION
    const result = await runInTransaction(queryRunner.manager)   // 사용자 쿼리들
    await queryRunner.commitTransaction()                        // COMMIT

    await queryRunner.release()                                  // 커넥션 반환
}
```

사용자 코드:

```typescript
await dataSource.transaction(async (manager) => {
    await manager.save(user1) // ┐
    await manager.save(user2) // ├─ 전부 같은 QueryRunner → 같은 커넥션
    await manager.remove(user3) // ┘
})
```

```
커넥션 풀: [conn1, conn2, conn3, ...]
                    │
                    ▼ (빌림)
QueryRunner ─── conn2 (고정)
  │
  ├─ "START TRANSACTION"     ← conn2
  ├─ "INSERT INTO user ..."  ← conn2
  ├─ "INSERT INTO user ..."  ← conn2
  ├─ "DELETE FROM user ..."  ← conn2
  ├─ "COMMIT"                ← conn2
  │
  └─ release()               → conn2를 풀에 반환
```

### 트랜잭션 없이 save()를 호출하면

```typescript
await repo.save(user1) // QueryRunner A → conn1 빌림 → INSERT → commit → conn1 반환
await repo.save(user2) // QueryRunner B → conn2 빌림 → INSERT → commit → conn2 반환
```

각 `save()`가 **별도의 QueryRunner = 별도의 커넥션 = 별도의 트랜잭션**이다.
첫 번째가 성공하고 두 번째가 실패해도 첫 번째는 이미 커밋된 상태다.

### JPA와 비교

|                      | JPA                                                 | TypeORM                                                          |
| -------------------- | --------------------------------------------------- | ---------------------------------------------------------------- |
| **커넥션 관리 주체** | Session이 내부적으로 관리                           | QueryRunner가 관리                                               |
| **같은 커넥션 보장** | Session이 보장                                      | QueryRunner가 `databaseConnection` 고정으로 보장                 |
| **트랜잭션 시작**    | `em.getTransaction().begin()` 또는 `@Transactional` | `queryRunner.startTransaction()` 또는 `dataSource.transaction()` |
| **자동 트랜잭션**    | Spring `@Transactional`이 자동화                    | `save()` 하나당 자동 또는 `transaction()` 수동                   |
