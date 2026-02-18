# TypeORM의 영속성 컨텍스트 부재 (JPA 비교)

## 개요

JPA의 핵심은 **영속성 컨텍스트(Persistence Context)** 이다.
1차 캐시(Identity Map), 쓰기 지연(Write-Behind), dirty checking이 모두 영속성 컨텍스트 위에서 동작한다.
TypeORM에는 이에 대응하는 개념이 없다.

---

## 1. Identity Map (1차 캐시)

### JPA: 같은 객체 보장

```java
// 같은 트랜잭션 안에서
User user1 = em.find(User.class, 1L);
User user2 = em.find(User.class, 1L);

System.out.println(user1 == user2);  // true — 같은 객체
```

영속성 컨텍스트가 PK를 키로 엔티티 인스턴스를 캐싱한다.
두 번째 조회 시 DB에 가지 않고 캐시된 인스턴스를 반환한다.

### TypeORM: 매번 새 객체

```typescript
await dataSource.transaction(async (manager) => {
    const user1 = await manager.findOne(User, { where: { id: 1 } })
    const user2 = await manager.findOne(User, { where: { id: 1 } })

    console.log(user1 === user2) // false — 다른 객체
})
```

#### 코드 근거

`SelectQueryBuilder.getOne()` (`src/query-builder/SelectQueryBuilder.ts:1788`):

```typescript
async getOne() {
    const results = await this.getRawAndEntities()  // 매번 SQL 실행
    return results.entities[0]
}
```

`RawSqlResultsToEntityTransformer.transform()` (`src/query-builder/transformer/RawSqlResultsToEntityTransformer.ts:63-70`):

```typescript
transform(rawResults, alias) {
    const group = this.group(rawResults, alias)  // PK로 그룹핑 (단일 쿼리 내 중복 제거용)
    for (const results of group.values()) {
        const entity = this.transformRawResultsGroup(results, alias)
        //              ↑ 내부에서 metadata.create() → 매번 새 인스턴스
    }
}
```

`group()`은 **한 쿼리 안에서** JOIN 때문에 중복된 row를 PK 기준으로 합치는 역할이지,
**쿼리 간** 동일 엔티티를 추적하는 Identity Map이 아니다.

조회 흐름:

```
findOne(id: 1)  →  SQL 실행  →  metadata.create()  →  새 인스턴스 A
findOne(id: 1)  →  SQL 실행  →  metadata.create()  →  새 인스턴스 B
                                                        A !== B
```

EntityManager에도, QueryRunner에도, 이전 조회 결과를 PK로 저장해두는 맵이 없다.

---

## 2. 쓰기 지연 (Write-Behind)

### JPA: SQL을 모아두었다가 flush 시 일괄 실행

```java
tx.begin();

em.persist(user1);    // SQL 실행 안 함 — 쓰기 지연 저장소에 INSERT 예약
em.persist(user2);    // SQL 실행 안 함 — 쓰기 지연 저장소에 INSERT 예약
user1.setName("Tom"); // SQL 실행 안 함 — dirty checking으로 UPDATE 예약

tx.commit();          // 이 시점에 INSERT 2개 + UPDATE 1개 한꺼번에 flush
```

영속성 컨텍스트가 SQL을 모아두었다가 `flush()` 또는 `commit()` 시점에 일괄 실행한다.
이를 통해 네트워크 왕복을 줄이고 batch insert 최적화가 가능하다.

### TypeORM: save() 호출 즉시 실행

```typescript
await manager.save(user1) // 즉시 INSERT 실행 + commit
await manager.save(user2) // 즉시 INSERT 실행 + commit
user1.name = "Tom"
await manager.save(user1) // 즉시 UPDATE 실행 + commit
```

#### 코드 근거

`EntityManager.save()` → `EntityPersistExecutor.execute()` (`src/persistence/EntityPersistExecutor.ts:41-206`):

```typescript
async execute() {
    // 1. Subject(변경 대상) 수집
    // 2. 캐스케이드 빌드
    // 3. DB에서 기존 엔티티 로드 (변경 비교용)

    // 4. 트랜잭션 시작 (없으면 자동 시작)
    await queryRunner.startTransaction()

    // 5. 즉시 실행
    for (const executor of executorsWithExecutableOperations) {
        await executor.execute()    // ← 바로 SQL 실행
    }

    // 6. 즉시 커밋
    await queryRunner.commitTransaction()
}
```

`SubjectExecutor.execute()` (`src/persistence/SubjectExecutor.ts:102-178`) — 순서대로 즉시 실행:

```typescript
async execute() {
    // 이벤트 브로드캐스트 (BeforeInsert, BeforeUpdate 등)

    await this.executeInsertOperations()      // INSERT 즉시 실행
    await this.executeUpdateOperations()      // UPDATE 즉시 실행
    await this.executeRemoveOperations()      // DELETE 즉시 실행
    await this.executeSoftRemoveOperations()  // soft DELETE 즉시 실행
    await this.executeRecoverOperations()     // recover 즉시 실행
}
```

모아뒀다가 한번에 보내는 것이 아니라, `save()` 한 번 = SQL 즉시 실행 + 커밋이다.

### 여러 엔티티를 한 트랜잭션으로 묶으려면

```typescript
// 방법 1: transaction 블록
await dataSource.transaction(async (manager) => {
    await manager.save(user1) // INSERT — 커밋은 아직
    await manager.save(user2) // INSERT — 커밋은 아직
    // 블록 끝에서 commit
})

// 방법 2: 배열로 한번에
await manager.save([user1, user2]) // 하나의 트랜잭션에서 INSERT 2개
```

하지만 이것은 JPA의 쓰기 지연과 다르다.
JPA는 **영속성 컨텍스트가 알아서** SQL을 모으지만, TypeORM은 **개발자가 명시적으로** 묶어줘야 한다.

---

## 3. Dirty Checking

### JPA: 자동 감지

```java
User user = em.find(User.class, 1L);  // 스냅샷 저장
user.setName("Tom");                   // 변경만 하면 됨
// tx.commit() 시 스냅샷과 비교 → 변경 감지 → UPDATE 자동 실행
```

영속성 컨텍스트가 엔티티 로드 시점에 스냅샷을 저장하고,
flush 시점에 현재 상태와 비교하여 변경된 필드만 UPDATE한다.

### TypeORM: save() 시 DB 재조회 후 비교

```typescript
const user = await manager.findOne(User, { where: { id: 1 } })
user.name = "Tom"
await manager.save(user) // 이 안에서 DB에서 다시 조회하여 비교
```

`EntityPersistExecutor.execute()` 안에서:

```typescript
// DB에서 기존 엔티티를 다시 로드하여 비교
await new SubjectDatabaseEntityLoader(queryRunner, subjects).load(this.mode)
```

TypeORM은 메모리에 스냅샷을 유지하지 않으므로, `save()` 호출 시 DB에서 현재 상태를 다시 조회한 뒤 비교한다.
또한 프로퍼티를 변경하는 것만으로는 아무 일도 일어나지 않고, 반드시 `save()`를 명시적으로 호출해야 한다.

---

## 4. AsyncLocalStorage로 영속성 컨텍스트를 구현할 수 있는가?

### 아이디어

JPA는 `ThreadLocal`로 요청별 영속성 컨텍스트를 바인딩한다.
Node.js에는 `AsyncLocalStorage`(ThreadLocal의 Node.js 대응)가 있으므로, 같은 구조를 만들 수 있다.

```
JPA:
  Thread → ThreadLocal → EntityManager → 영속성 컨텍스트

Node.js에서 같은 구조:
  Request → AsyncLocalStorage → 커스텀 Context → Identity Map, 쓰기 지연 큐
```

```typescript
// 가상의 구현
import { AsyncLocalStorage } from "async_hooks"

const als = new AsyncLocalStorage<PersistenceContext>()

// 요청마다 새 컨텍스트 생성
app.use((req, res, next) => {
    als.run(new PersistenceContext(), next)
})

// findOne() — Identity Map 먼저 확인
async findOne(User, id) {
    const ctx = als.getStore()
    const cached = ctx.identityMap.get(User, id)
    if (cached) return cached          // DB 안 감

    const entity = await queryDB(...)
    ctx.identityMap.set(User, id, entity)
    return entity
}

// save() — 즉시 실행 대신 큐에 모음
async save(entity) {
    const ctx = als.getStore()
    ctx.writeQueue.push(entity)        // 쓰기 지연
}

// flush — 한꺼번에 실행
async flush() {
    const ctx = als.getStore()
    await executeBatch(ctx.writeQueue)  // dirty checking + batch SQL
}
```

### 실제로 이렇게 만든 ORM: MikroORM

MikroORM은 **Node.js에서 JPA의 영속성 컨텍스트를 실제로 구현한 ORM**이다.

```typescript
// MikroORM — JPA와 거의 같은 패턴
const em = orm.em.fork() // 요청마다 새 EntityManager

const user1 = await em.findOne(User, 1)
const user2 = await em.findOne(User, 1)
console.log(user1 === user2) // true — Identity Map 동작!

user1.name = "Tom"
// save() 호출 안 해도 됨 — dirty checking이 자동 감지

await em.flush() // 이 시점에 UPDATE 실행 (쓰기 지연)
```

MikroORM의 `RequestContext`가 내부적으로 AsyncLocalStorage를 사용한다:

```typescript
// MikroORM의 미들웨어
app.use((req, res, next) => {
    RequestContext.create(orm.em, next) // AsyncLocalStorage.run() 내부 호출
})
```

### TypeORM에 추가하기 어려운 이유

기술적으로 AsyncLocalStorage를 쓰면 가능하지만, TypeORM의 **전체 설계를 바꿔야** 한다:

| 구성 요소          | 현재 TypeORM                | 영속성 컨텍스트 추가 시 |
| ------------------ | --------------------------- | ----------------------- |
| **EntityManager**  | stateless, 앱에 1개         | stateful, 요청마다 생성 |
| **findOne()**      | 항상 SQL 실행               | Identity Map 먼저 확인  |
| **save()**         | 즉시 INSERT/UPDATE          | 쓰기 지연 큐에 추가     |
| **dirty checking** | save() 때 DB 재조회 후 비교 | 스냅샷과 자동 비교      |
| **flush**          | 개념 없음                   | 큐의 SQL 일괄 실행      |

EntityManager가 stateless라는 것이 TypeORM의 근본 설계이므로, 영속성 컨텍스트를 "추가"하는 것이 아니라 사실상 **다른 ORM을 만드는 것**에 가깝다.

### Node.js ORM 비교: 영속성 컨텍스트 기준

|                            | JPA (Hibernate)  | MikroORM          | TypeORM             |
| -------------------------- | ---------------- | ----------------- | ------------------- |
| **요청별 컨텍스트 바인딩** | ThreadLocal      | AsyncLocalStorage | 없음                |
| **Identity Map**           | 있음             | 있음              | 없음                |
| **쓰기 지연**              | 있음             | 있음              | 없음                |
| **dirty checking**         | 스냅샷 자동 비교 | 스냅샷 자동 비교  | save() 때 DB 재조회 |
| **flush**                  | 있음             | 있음              | 없음                |

**결론:** AsyncLocalStorage라는 도구는 맞지만, TypeORM의 핵심 설계(stateless EntityManager)를 전면 수정해야 한다. 영속성 컨텍스트가 필요하면 MikroORM이 Node.js에서 가장 가까운 선택지다.

---

## 비교 요약

|                        | JPA (Hibernate)                       | TypeORM                                            |
| ---------------------- | ------------------------------------- | -------------------------------------------------- |
| **영속성 컨텍스트**    | 있음 (EntityManager = 1차 캐시)       | 없음                                               |
| **Identity Map**       | 같은 트랜잭션에서 동일 PK → 같은 객체 | 매번 새 객체                                       |
| **2차 조회 시 SQL**    | 실행 안 함 (캐시 히트)                | 매번 실행                                          |
| **쓰기 지연**          | 있음 — flush 시점에 일괄 실행         | 없음 — `save()` 즉시 실행                          |
| **flush**              | `commit()`, `flush()`, JPQL 실행 전   | 개념 없음 (`save()` = 즉시 flush)                  |
| **batch insert**       | flush 시 모아서 batch 가능            | `save([...])` 배열로 넘기면 한 트랜잭션 처리       |
| **dirty checking**     | 자동 — 스냅샷 비교                    | `save()` 때 DB 재조회 후 비교                      |
| **변경 감지 트리거**   | `commit()` / `flush()` 자동           | `save()` 명시적 호출 필수                          |
| **Query Result Cache** | 2차 캐시 (선택적)                     | `cache()` 옵션 (쿼리 결과 캐싱, Identity Map 아님) |

---

## 관련 소스 파일

| 파일                                                                | 역할                                                      |
| ------------------------------------------------------------------- | --------------------------------------------------------- |
| `src/entity-manager/EntityManager.ts`                               | `save()`, `findOne()` 등 진입점                           |
| `src/persistence/EntityPersistExecutor.ts`                          | `save()` 실행 — Subject 수집, 트랜잭션, 즉시 실행         |
| `src/persistence/SubjectExecutor.ts`                                | INSERT/UPDATE/DELETE 순서대로 즉시 실행                   |
| `src/persistence/SubjectDatabaseEntityLoader.ts`                    | `save()` 시 DB에서 기존 상태 재조회 (dirty checking 대체) |
| `src/query-builder/transformer/RawSqlResultsToEntityTransformer.ts` | 조회 시 매번 `metadata.create()` → 새 인스턴스            |
