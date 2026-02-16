# TypeORM Lazy Relation 내부 동작 원리 (JPA 비교)

## 개요

TypeORM의 lazy relation은 엔티티의 관계 프로퍼티에 접근하는 시점에 DB 쿼리를 실행하는 지연 로딩 메커니즘이다.
JPA가 바이트코드 조작(프록시 서브클래스)으로 구현하는 것과 달리, TypeORM은 JavaScript의 `Object.defineProperty`와 클로저를 활용한다.

---

## JPA vs TypeORM 비교

| | JPA (Hibernate) | TypeORM |
|---|---|---|
| **지연 로딩 구현** | 바이트코드 조작으로 프록시 서브클래스 생성 | `Object.defineProperty`로 getter 교체 |
| **프록시 타입** | `User$HibernateProxy` (서브클래스) | 원본 클래스 인스턴스 그대로 |
| **lazy 판정** | `@ManyToOne(fetch = FetchType.LAZY)` | 프로퍼티 타입이 `Promise<T>`이면 자동 판정 |
| **트리거** | 프록시 객체의 메서드 호출 시 | 프로퍼티 접근(`user.posts`) 시 getter 실행 |
| **세션/커넥션 의존** | EntityManager(Session) 닫히면 `LazyInitializationException` | DataSource 클로저 캡처, destroy 후에도 참조는 남음 |
| **메타데이터 시점** | 앱 시작 시 SessionFactory 빌드 | `DataSource.initialize()` 시 빌드 |

---

## 전체 흐름

### Phase 1: 데코레이터 실행 — 원재료 수집

**시점: `import`로 엔티티 클래스가 로드될 때**

```typescript
// 이 파일을 import하는 순간 데코레이터가 실행됨
@Entity()
class User {
    @PrimaryColumn()
    id: number

    @OneToMany(() => Post, (post) => post.user)
    posts: Promise<Post[]>   // Promise 타입 → lazy relation
}
```

`@OneToMany` 데코레이터 (`src/decorator/relations/OneToMany.ts:11-47`):

```typescript
export function OneToMany(...) {
    return function (object, propertyName) {
        // Promise 타입이면 자동으로 isLazy = true 판정
        const reflectedType = Reflect.getMetadata("design:type", object, propertyName)
        if (reflectedType.name.toLowerCase() === "promise")
            isLazy = true

        // 전역 저장소에 원재료(args)를 push
        getMetadataArgsStorage().relations.push({
            target: User,
            propertyName: "posts",
            relationType: "one-to-many",
            type: () => Post,
            inverseSideProperty: (post) => post.user,
            isLazy: true,
        })
    }
}
```

전역 저장소는 프로세스 전체에서 하나의 `MetadataArgsStorage`를 공유한다 (`src/globals.ts:23-37`).

> **JPA 비교:** JPA도 앱 시작 시 `@OneToMany` 등의 어노테이션을 스캔하여 메타데이터를 수집한다.
> 차이점은 JPA는 리플렉션으로 어노테이션을 읽지만, TypeORM은 데코레이터 함수가 직접 전역 저장소에 push한다.

---

### Phase 2: `DataSource.initialize()` — 메타데이터 빌드 + prototype에 getter 설치

**시점: `await dataSource.initialize()` 호출 시**

#### 2-1. 원재료 → RelationMetadata 가공

```
DataSource.initialize()
  → ConnectionMetadataBuilder.buildEntityMetadatas()     // src/data-source/DataSource.ts:753
    → EntityMetadataBuilder.build()                      // src/connection/ConnectionMetadataBuilder.ts:103
```

1단계에서 쌓인 원재료(args)를 가공하여 완성된 메타데이터 객체를 만든다:

```
원재료 (args)                        가공된 메타데이터
─────────────                       ──────────────────
{ target: User,                →    RelationMetadata {
  propertyName: "posts",              propertyName: "posts",
  relationType: "one-to-many",        isOneToMany: true,
  type: () => Post,                   inverseRelation: Post→User의 ManyToOne,
  isLazy: true }                      joinColumns: [{ propertyPath: "userId",
                                          referencedColumn: User.id }],
                                      isLazy: true,
                                      entityMetadata: User의 EntityMetadata,
                                    }
```

양쪽 관계를 서로 연결하고(`inverseRelation`), joinColumn 정보를 계산하는 등 단순 args에서는 알 수 없었던 관계 구조를 완성한다.

> **JPA 비교:** Hibernate의 `SessionFactory` 빌드 과정과 동일한 역할.
> Hibernate도 이 시점에 `@JoinColumn`, 양방향 관계의 `mappedBy` 등을 해석하여 매핑 메타데이터를 완성한다.

#### 2-2. prototype에 lazy getter 설치

메타데이터 빌드 마지막 단계 (`src/metadata-builder/EntityMetadataBuilder.ts:397-409`):

```typescript
entityMetadatas
    .filter((metadata) => typeof metadata.target === "function")
    .forEach((entityMetadata) => {
        entityMetadata.relations
            .filter((relation) => relation.isLazy)
            .forEach((relation) => {
                this.connection.relationLoader.enableLazyLoad(
                    relation,
                    (entityMetadata.target as Function).prototype,  // User.prototype에 설치
                    // queryRunner 없음!
                )
            })
    })
```

이 getter는 `new User()`로 직접 인스턴스를 만들었을 때의 fallback용이다.
queryRunner가 없으므로, 호출 시 새 커넥션을 잡아서 쿼리한다.

---

### Phase 3: `findOne()` 등 쿼리 실행 — 인스턴스에 getter 설치

**시점: `await userRepo.findOne({ where: { id: 1 } })` 호출 시**

#### 3-1. 쿼리 실행 흐름

```
Repository.findOne()                                    // src/repository/Repository.ts:633
  → EntityManager.findOne()                             // src/entity-manager/EntityManager.ts:1306
    → SelectQueryBuilder.getOne()                       // src/query-builder/SelectQueryBuilder.ts:1788
      → getRawAndEntities()                             // :1745
        → executeEntitiesAndRawResults(queryRunner)     // :3539
          → SQL 실행, rawResults 획득
          → RawSqlResultsToEntityTransformer.transform()  // :3734
```

#### 3-2. raw 결과 → 엔티티 변환 (hydration)

`RawSqlResultsToEntityTransformer` (`src/query-builder/transformer/RawSqlResultsToEntityTransformer.ts:200`):

```typescript
const entity = metadata.create(this.queryRunner, {
    fromDeserializer: true,
})
```

#### 3-3. `EntityMetadata.create()` — 인스턴스 생성 + lazy getter 설치

`src/metadata/EntityMetadata.ts:563-593`:

```typescript
create(queryRunner?, options?) {
    // 인스턴스 생성
    let ret = Object.create(this.target.prototype)  // 또는 new this.target()

    // 모든 lazy relation에 대해 getter/setter 설치
    this.lazyRelations.forEach((relation) =>
        this.connection.relationLoader.enableLazyLoad(
            relation,
            ret,          // 이 인스턴스에 직접 설치
            queryRunner,  // 현재 쿼리의 queryRunner 포함!
        )
    )
    return ret
}
```

> **JPA 비교:** Hibernate는 이 시점에 프록시 서브클래스의 인스턴스를 만든다.
> `User$HibernateProxy`가 `User`를 상속하며, getter 메서드를 오버라이드하여 최초 접근 시 SQL을 실행한다.
> TypeORM은 서브클래스 대신 원본 인스턴스의 프로퍼티를 `Object.defineProperty`로 교체하는 방식이다.

---

### Phase 4: `RelationLoader.enableLazyLoad()` — Object.defineProperty의 실제 동작

`src/query-builder/RelationLoader.ts:386-455`:

```typescript
enableLazyLoad(relation, entity, queryRunner?) {
    const relationLoader = this   // RelationLoader 인스턴스 (→ DataSource 참조)

    Object.defineProperty(entity, relation.propertyName, {
        get: function () {
            // ── 1. 이미 로드 완료 → 캐시 반환 ──
            if (this[resolveIndex] === true || this[dataIndex] !== undefined)
                return Promise.resolve(this[dataIndex])

            // ── 2. 로딩 중 → 기존 Promise 반환 (중복 쿼리 방지) ──
            if (this[promiseIndex])
                return this[promiseIndex]

            // ── 3. 최초 접근 → DB 쿼리 실행 ──
            const loader = relationLoader
                .load(relation, this, queryRunner)
                .then(...)
            return setPromise(this, loader)
        },
        set: function (value) { ... },
        configurable: true,
        enumerable: false,
    })
}
```

#### getter 내부의 세 가지 분기

```
user.posts (1번째 접근)
  → 분기 3: DB 쿼리 시작, Promise를 __promise_posts__에 저장

user.posts (쿼리 완료 전 재접근)
  → 분기 2: __promise_posts__에 있는 동일 Promise 반환 (중복 쿼리 방지)

  ... DB 응답 도착 → __posts__에 결과 저장, __has_posts__ = true

user.posts (이후 접근)
  → 분기 1: __posts__에서 캐시된 결과 즉시 반환
```

#### getter 안의 `this`

getter는 `function() {}` 형태이므로 호출 시 점(.) 앞의 객체가 `this`가 된다:

```typescript
user.posts   // user.에 의해 getter 실행 → this = user
```

반면 `relationLoader`는 `enableLazyLoad` 실행 시점에 클로저로 캡처된 것:

```typescript
enableLazyLoad(...) {
    const relationLoader = this   // ← 여기서 this = RelationLoader (메서드 호출이므로)
    Object.defineProperty(entity, ..., {
        get: function () {
            relationLoader  // ← 클로저로 기억한 RelationLoader
            this            // ← getter의 this = entity (user)
        },
    })
}
```

---

### Phase 5: lazy relation 접근 — 실제 쿼리 생성

`user.posts` 접근 시 `relationLoader.load(relation, this, queryRunner)` 가 호출된다.

`relation`(메타데이터)과 `this`(entity)가 조합되어 쿼리가 만들어진다:

```
relation (메타데이터) 제공:
  ├─ 관계 종류: OneToMany
  ├─ 대상 테이블: Post
  └─ 연결 기준: Post.userId = User.id  (쿼리의 구조)

this (entity 인스턴스) 제공:
  └─ id: 42                             (쿼리의 파라미터 값)

→ SELECT * FROM post WHERE userId IN (42)
```

관계 종류에 따라 `RelationLoader`의 다른 메서드가 호출된다 (`src/query-builder/RelationLoader.ts:26-64`):

| 관계 종류 | 메서드 |
|---|---|
| ManyToOne / OneToOne (owner) | `loadManyToOneOrOneToOneOwner()` |
| OneToMany / OneToOne (non-owner) | `loadOneToManyOrOneToOneNotOwner()` |
| ManyToMany (owner) | `loadManyToManyOwner()` |
| ManyToMany (non-owner) | `loadManyToManyNotOwner()` |

---

## prototype vs 인스턴스 getter

`enableLazyLoad`는 두 번 호출된다:

| | Phase 2 (prototype) | Phase 3 (인스턴스) |
|---|---|---|
| **시점** | `dataSource.initialize()` | `findOne()` 등 쿼리 시 |
| **대상** | `User.prototype` | 개별 entity 인스턴스 |
| **queryRunner** | `undefined` (새 커넥션) | 현재 쿼리의 queryRunner |
| **용도** | `new User()`로 직접 만든 경우의 fallback | 쿼리로 hydrate된 경우 (트랜잭션 유지) |
| **우선순위** | 낮음 | 높음 (인스턴스 프로퍼티가 prototype보다 우선) |

**JavaScript의 프로퍼티 탐색 순서:**

```
user.posts 를 읽는다
  ├─ 1. user 자신에 "posts"가 있나? → 있으면 사용 (인스턴스 getter)
  └─ 2. 없으면 User.prototype에서 탐색 (prototype getter)
```

- `findOne()`으로 가져온 엔티티: 인스턴스에 설치됨 → queryRunner 포함 → 트랜잭션 안에서 일관성 보장
- `new User()`로 직접 만든 엔티티: 인스턴스에 없음 → prototype의 것 사용 → 새 커넥션으로 쿼리

---

## 클로저에 의한 DataSource 참조 체인

```
entity (User 인스턴스)
  └─ .posts (getter 함수)
       └─ 클로저로 캡처:
            ├─ relationLoader ──→ RelationLoader 인스턴스
            │                       └─ .connection ──→ DataSource (driver, pool, 전체 metadata 등)
            ├─ relation        ──→ RelationMetadata
            └─ queryRunner     ──→ QueryRunner (또는 undefined)
```

**의미:**
- 엔티티 인스턴스가 메모리에 살아있는 한, DataSource 전체가 GC되지 못한다.
- `DataSource.destroy()` 후에도 이미 생성된 엔티티의 getter는 파괴된 DataSource를 여전히 참조한다.

> **JPA 비교:** Hibernate에서도 프록시가 Session을 참조하지만,
> Session이 닫히면 `LazyInitializationException`을 명시적으로 던진다.
> TypeORM은 이런 예외 없이 죽은 DataSource로 쿼리를 시도할 수 있어 더 위험할 수 있다.

---

## 관련 소스 파일 요약

| 파일 | 역할 |
|---|---|
| `src/decorator/relations/OneToMany.ts` | `@OneToMany` 데코레이터, 전역 저장소에 args push |
| `src/globals.ts` | 전역 `MetadataArgsStorage` 싱글턴 |
| `src/metadata-builder/EntityMetadataBuilder.ts` | args → `RelationMetadata` 빌드, prototype에 getter 설치 |
| `src/metadata/EntityMetadata.ts` | `create()` — 인스턴스 생성 + 인스턴스에 getter 설치 |
| `src/metadata/RelationMetadata.ts` | 관계 메타데이터 (joinColumn, inverseRelation 등) |
| `src/query-builder/RelationLoader.ts` | `enableLazyLoad()` — `Object.defineProperty` 실행, `load()` — 실제 쿼리 |
| `src/query-builder/transformer/RawSqlResultsToEntityTransformer.ts` | raw SQL 결과 → 엔티티 hydration |
| `src/query-builder/SelectQueryBuilder.ts` | `getOne()` → `executeEntitiesAndRawResults()` |
| `src/data-source/DataSource.ts` | `RelationLoader` 생성, `initialize()` 진입점 |