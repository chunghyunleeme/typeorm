# 기본 키 매핑 (JPA vs TypeORM)

## 개요

JPA는 `@Id` + `@GeneratedValue`로 기본 키를 매핑한다.
TypeORM은 `@PrimaryColumn()` 또는 `@PrimaryGeneratedColumn()`으로 매핑한다.
JPA는 두 어노테이션을 조합하지만, TypeORM은 하나의 데코레이터로 합쳤다.

---

## 1. 데코레이터 비교

### JPA

```java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
}
```

- `@Id` — 이 필드가 PK임을 선언
- `@GeneratedValue` — 값 생성 전략 지정

### TypeORM

```typescript
@Entity()
class User {
    @PrimaryGeneratedColumn()
    id: number
}
```

| TypeORM 데코레이터                    | JPA 대응                              | 설명                         |
| ------------------------------------- | ------------------------------------- | ---------------------------- |
| `@PrimaryColumn()`                    | `@Id`                                 | 수동 할당 PK                 |
| `@PrimaryGeneratedColumn()`           | `@Id` + `@GeneratedValue(IDENTITY)`   | auto_increment               |
| `@PrimaryGeneratedColumn("uuid")`     | `@Id` + `@GeneratedValue` + UUID 전략 | UUID 자동 생성               |
| `@PrimaryGeneratedColumn("identity")` | `@Id` + `@GeneratedValue(IDENTITY)`   | PostgreSQL 10+ IDENTITY 컬럼 |
| `@PrimaryGeneratedColumn("rowid")`    | —                                     | CockroachDB 전용             |

---

## 2. 생성 전략 비교

### JPA의 GenerationType

| 전략       | 설명                     | INSERT 전 ID 확보                 |
| ---------- | ------------------------ | --------------------------------- |
| `IDENTITY` | DB auto_increment        | 불가 — INSERT해야 ID를 안다       |
| `SEQUENCE` | DB 시퀀스 사용           | **가능** — 시퀀스만 조회하면 된다 |
| `TABLE`    | 별도 키 생성 테이블      | **가능** — 테이블에서 조회        |
| `AUTO`     | DB 방언에 따라 자동 선택 | 전략에 따라 다름                  |

### TypeORM의 생성 전략

| 전략                 | 대응 JPA 전략 | INSERT 전 ID 확보          |
| -------------------- | ------------- | -------------------------- |
| `"increment"` (기본) | `IDENTITY`    | 불가 — DB가 생성           |
| `"uuid"`             | —             | DB에 따라 다름 (아래 참고) |
| `"identity"`         | `IDENTITY`    | 불가 — DB가 생성           |
| `"rowid"`            | —             | 불가 — DB가 생성           |

TypeORM에는 **SEQUENCE 전략이 없다**.

---

## 3. 데코레이터 내부 동작

### `@PrimaryGeneratedColumn()` — `src/decorator/columns/PrimaryGeneratedColumn.ts`

```typescript
export function PrimaryGeneratedColumn(strategyOrOptions?, maybeOptions?) {
    let strategy = "increment" // 기본 전략
    if (typeof strategyOrOptions === "string") strategy = strategyOrOptions

    return function (object, propertyName) {
        // 전략에 따라 컬럼 타입 자동 결정
        if (strategy === "increment" || strategy === "identity")
            options.type = Number
        else if (strategy === "uuid") options.type = "uuid"

        options.primary = true

        // 글로벌 저장소에 두 가지 메타데이터를 따로 등록
        getMetadataArgsStorage().columns.push({
            target: object.constructor,
            propertyName,
            mode: "regular",
            options,
        })
        getMetadataArgsStorage().generations.push({
            target: object.constructor,
            propertyName,
            strategy, // "increment" | "uuid" | "rowid" | "identity"
        })
    }
}
```

핵심: 하나의 데코레이터가 **두 개의 메타데이터**를 등록한다.

- `columns` — "이 프로퍼티는 PK 컬럼이다"
- `generations` — "이 컬럼은 이 전략으로 값을 생성한다"

### `@PrimaryColumn()` — `src/decorator/columns/PrimaryColumn.ts`

```typescript
// 수동 할당 PK — generation 메타데이터를 등록하지 않음
options.primary = true

getMetadataArgsStorage().columns.push({ ... })

// options.generated가 명시된 경우에만 generation 등록
if (options.generated) {
    getMetadataArgsStorage().generations.push({ ... })
}
```

### 메타데이터 빌드 시점 (initialize)

`EntityMetadataBuilder.build()`에서 두 메타데이터가 합쳐진다:

```typescript
entityMetadata.columns.forEach((column) => {
    const generated = this.metadataArgsStorage.findGenerated(
        column.target,
        column.propertyName,
    )
    if (generated) {
        column.isGenerated = true
        column.generationStrategy = generated.strategy
    }
})
```

---

## 4. UUID 생성: DB vs 앱

`InsertQueryBuilder.ts:1565-1577`:

```typescript
if (
    column.isGenerated &&
    column.generationStrategy === "uuid" &&
    !this.connection.driver.isUUIDGenerationSupported() && // DB가 UUID 생성 불가?
    value === undefined
) {
    value = RandomGenerator.uuidv4() // 앱에서 직접 UUID 생성
}
```

| DB                                  | `isUUIDGenerationSupported()` | UUID 생성 위치  | INSERT SQL                                      |
| ----------------------------------- | ----------------------------- | --------------- | ----------------------------------------------- |
| PostgreSQL, SQL Server, CockroachDB | `true`                        | **DB**          | `DEFAULT` (gen_random_uuid() 등)                |
| MySQL, SQLite, Oracle               | `false`                       | **앱(TypeORM)** | `RandomGenerator.uuidv4()` 값을 파라미터로 전달 |

두 경우 모두 `save()` **안에서** 일어나므로, `save()` 호출 **전에** 애플리케이션 코드에서 UUID를 알 수는 없다.

---

## 5. INSERT 전에 식별자가 필요한 경우

### 문제 상황: 트랜잭션 안에서 ID로 이벤트 발행

```java
// JPA (SEQUENCE 전략)
@Transactional
public void createUser(String name) {
    User user = new User(name);
    em.persist(user);
    // INSERT 안 됨, 시퀀스만 조회 → ID 확보

    eventPublisher.publish(new UserCreatedEvent(user.getId()));  // ✅ ID 사용 가능

    // 메서드 끝 → flush → INSERT → commit
}
```

```
persist()  →  SELECT nextval  →  ID 확보  →  이벤트 발행  →  flush  →  INSERT  →  commit
               ↑ 가벼운 조회                                           ↑ 실제 INSERT
```

SEQUENCE 전략은 가벼운 시퀀스 조회만으로 ID를 확보하고, INSERT는 flush 시점에 batch로 모아서 실행할 수 있다.

### TypeORM: save() 후에 ID 확보

```typescript
await dataSource.transaction(async (manager) => {
    const user = await manager.save(new User({ name: "Tom" }))
    // save() = 즉시 INSERT 실행 (commit은 아직)
    // user.id에 값이 들어있다

    eventPublisher.publish(new UserCreatedEvent(user.id)) // ✅ ID 사용 가능
})
// 블록 끝 → commit
```

```
save()  →  INSERT 실행  →  ID 확보  →  이벤트 발행  →  commit
            ↑ 즉시 실행
```

기능적으로는 **둘 다 트랜잭션 안에서 ID를 사용할 수 있다**. 차이는 INSERT 시점이다.

|              | JPA (SEQUENCE)              | TypeORM                   |
| ------------ | --------------------------- | ------------------------- |
| ID 확보 시점 | `persist()` — 시퀀스 조회만 | `save()` — INSERT 실행 후 |
| INSERT 시점  | flush/commit                | `save()` 즉시             |
| batch INSERT | 가능 (flush 시 모아서)      | 불가                      |

### 트랜잭션 롤백 시 이벤트 문제

```typescript
await dataSource.transaction(async (manager) => {
    const user = await manager.save(new User({ name: "Tom" }))

    eventPublisher.publish(new UserCreatedEvent(user.id)) // 이벤트 나감!

    await manager.save(new Profile({ userId: user.id })) // 💥 실패
})
// 롤백 — 하지만 이벤트는 이미 발행됨
```

이 문제는 JPA든 TypeORM이든 **동일하게 발생**한다. 해결 방법:

#### 방법 1: Outbox 패턴 — 이벤트를 같은 트랜잭션에 저장

```typescript
await dataSource.transaction(async (manager) => {
    const user = await manager.save(new User({ name: "Tom" }))

    // 이벤트를 외부로 보내지 않고 DB에 함께 저장
    await manager.save(
        new OutboxEvent({
            aggregateId: user.id,
            type: "UserCreated",
            payload: JSON.stringify({ userId: user.id }),
        }),
    )
})
// commit 성공 → 별도 프로세스가 outbox 테이블 폴링 → 이벤트 발행
// rollback → outbox도 함께 롤백 → 이벤트 안 나감
```

#### 방법 2: commit 후 이벤트 발행

```typescript
const user = await dataSource.transaction(async (manager) => {
    const user = await manager.save(new User({ name: "Tom" }))
    await manager.save(new Profile({ userId: user.id }))
    return user
})
// commit 완료 후
eventPublisher.publish(new UserCreatedEvent(user.id)) // 안전
```

> **JPA 비교:** Spring의 `@TransactionalEventListener(phase = AFTER_COMMIT)`과 같은 역할.

---

## 6. 복합 키

```java
// JPA — @IdClass 필요
@Entity
@IdClass(OrderItemId.class)
public class OrderItem {
    @Id private Long orderId;
    @Id private Long itemId;
}

// 별도 ID 클래스 필요
public class OrderItemId implements Serializable {
    private Long orderId;
    private Long itemId;
}
```

```typescript
// TypeORM — @PrimaryColumn 여러 개
@Entity()
class OrderItem {
    @PrimaryColumn()
    orderId: number

    @PrimaryColumn()
    itemId: number
}
```

JPA는 `@IdClass` 또는 `@EmbeddedId`로 복합 키 클래스를 별도로 만들어야 하지만,
TypeORM은 `@PrimaryColumn()`을 여러 개 붙이면 된다.

---

## 비교 요약

|                          | JPA                                | TypeORM                                                      |
| ------------------------ | ---------------------------------- | ------------------------------------------------------------ |
| **PK 선언**              | `@Id`                              | `@PrimaryColumn()`                                           |
| **자동 생성**            | `@Id` + `@GeneratedValue` (2개)    | `@PrimaryGeneratedColumn()` (1개)                            |
| **SEQUENCE 전략**        | 기본 지원 — INSERT 전 ID 확보 가능 | 없음                                                         |
| **IDENTITY 전략**        | 지원 — INSERT 즉시 실행            | `"increment"` (기본값)                                       |
| **UUID**                 | 구현체별 확장                      | `"uuid"` — DB 또는 앱에서 생성                               |
| **복합 키**              | `@IdClass` / `@EmbeddedId` 필요    | `@PrimaryColumn()` 여러 개                                   |
| **INSERT 전 ID 필요 시** | SEQUENCE 전략 사용                 | 앱에서 직접 생성 (UUID, ULID 등)                             |
| **메타데이터**           | 어노테이션 → SessionFactory        | 데코레이터 → `MetadataArgsStorage` → `EntityMetadataBuilder` |

---

## 관련 소스 파일

| 파일                                              | 역할                                                   |
| ------------------------------------------------- | ------------------------------------------------------ |
| `src/decorator/columns/PrimaryColumn.ts`          | `@PrimaryColumn()` — 수동 할당 PK                      |
| `src/decorator/columns/PrimaryGeneratedColumn.ts` | `@PrimaryGeneratedColumn()` — 자동 생성 PK             |
| `src/metadata-args/GeneratedMetadataArgs.ts`      | 생성 전략 메타데이터 인터페이스                        |
| `src/metadata-args/MetadataArgsStorage.ts`        | 글로벌 메타데이터 저장소 (`columns`, `generations`)    |
| `src/metadata-builder/EntityMetadataBuilder.ts`   | `columns` + `generations` 합쳐서 `ColumnMetadata` 완성 |
| `src/query-builder/InsertQueryBuilder.ts:1565`    | UUID 생성 분기 (DB vs 앱)                              |
