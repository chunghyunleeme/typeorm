# TypeORM 구동 방식 (JPA 비교)

## 개요

JPA는 `persistence.xml` → `EntityManagerFactory` → `EntityManager` 순서로 구동된다.
TypeORM은 `DataSourceOptions` → `new DataSource()` → `initialize()` 순서로 구동된다.
두 구조는 비슷해 보이지만, EntityManager의 수명과 역할에서 근본적인 차이가 있다.

---

## JPA 구동 흐름

```
1. META-INF/persistence.xml          ← 설정 정보 (DB URL, 엔티티 목록 등)
         │
         ▼
2. EntityManagerFactory 생성          ← 무거운 초기화 (메타데이터 빌드, 커넥션 풀)
   Persistence.createEntityManagerFactory("unitName")
         │                               앱 전체에서 1개, 재사용
         ▼
3. EntityManager 생성                 ← 가벼운 생성 (요청마다 새로 만듦)
   emf.createEntityManager()             영속성 컨텍스트 = 1차 캐시 포함
```

- `EntityManagerFactory`는 앱 전체에서 **1개** — 무거운 초기화 (메타데이터 파싱, 커넥션 풀)
- `EntityManager`는 **요청(트랜잭션)마다** 새로 생성 — 영속성 컨텍스트(1차 캐시, 쓰기 지연) 포함

---

## TypeORM 구동 흐름

### Step 1: `new DataSource(options)` — 가벼운 생성

`src/data-source/DataSource.ts:141-160`:

```typescript
constructor(options: DataSourceOptions) {
    this.options = options
    this.driver = new DriverFactory().create(this)   // Driver 생성 (아직 연결 안 됨)
    this.manager = this.createEntityManager()         // EntityManager 생성
    this.relationLoader = new RelationLoader(this)    // lazy relation 로더
    this.isInitialized = false                        // 아직 초기화 안 됨
}
```

설정 정보(`DataSourceOptions`)는 XML이 아니라 JS 객체로 전달된다:

```typescript
const dataSource = new DataSource({
    type: "postgres",
    host: "localhost",
    port: 5432,
    username: "user",
    password: "pass",
    database: "mydb",
    entities: [User, Post], // 엔티티 클래스 직접 전달
    synchronize: true,
})
```

> **JPA 비교:** `persistence.xml`의 역할. JPA는 XML(또는 Spring Boot의 `application.yml`)에서 설정을 읽지만,
> TypeORM은 JS 객체로 직접 전달한다. 파일에서 읽을 수도 있지만(`ormconfig.json` 등) 필수는 아니다.

### Step 2: `dataSource.initialize()` — 무거운 초기화

`src/data-source/DataSource.ts:248-286`:

```typescript
async initialize() {
    // 1. DB 커넥션 풀 생성
    await this.driver.connect()

    // 2. 캐시 DB 연결 (옵션)
    if (this.queryResultCache) await this.queryResultCache.connect()

    // 3. 메타데이터 빌드 (핵심)
    await this.buildMetadatas()

    // 4. DDL 동기화 (옵션)
    if (this.options.synchronize) await this.synchronize()

    // 5. 마이그레이션 실행 (옵션)
    if (this.options.migrationsRun) await this.runMigrations()
}
```

> **JPA 비교:** `Persistence.createEntityManagerFactory("unitName")`의 역할.
> 커넥션 풀 생성 + 메타데이터 빌드가 모두 이 시점에 일어난다.

### `buildMetadatas()` 상세

`src/data-source/DataSource.ts:734-777`:

```typescript
protected async buildMetadatas() {
    const connectionMetadataBuilder = new ConnectionMetadataBuilder(this)

    // subscribers 빌드
    const subscribers = await connectionMetadataBuilder.buildSubscribers(...)

    // 엔티티 메타데이터 빌드 (RelationMetadata, ColumnMetadata 등 완성)
    const entityMetadatas = await connectionMetadataBuilder.buildEntityMetadatas(...)

    // migrations 빌드
    const migrations = await connectionMetadataBuilder.buildMigrations(...)

    // 메타데이터 검증 (관계 무결성, 컬럼 타입 등)
    entityMetadataValidator.validateMany(this.entityMetadatas, this.driver)
}
```

> **JPA 비교:** Hibernate의 `SessionFactory` 빌드 과정.
> `@Entity`, `@Column`, `@OneToMany` 등을 해석하여 매핑 메타데이터를 완성한다.

### 엔티티 인스턴스 생성: `EntityMetadata.create()`

조회 결과를 엔티티 객체로 변환할 때 `EntityMetadata.create()`가 호출된다 (`src/metadata/EntityMetadata.ts:563-593`):

```typescript
create(queryRunner?, options?) {
    let ret;
    if (typeof this.target === "function" && !pojo) {
        if (!options?.fromDeserializer || this.isAlwaysUsingConstructor) {
            ret = new (<any>this.target)()              // 생성자 호출
        } else {
            ret = Object.create(this.target.prototype)  // 생성자 우회
        }
    }
    // lazy relation getter 설치
    this.lazyRelations.forEach((relation) =>
        this.connection.relationLoader.enableLazyLoad(relation, ret, queryRunner)
    )
    return ret
}
```

두 가지 경로가 있다:

- `new this.target()` — 생성자를 호출하여 인스턴스 생성
- `Object.create(this.target.prototype)` — 생성자를 **우회**하여 프로토타입만 연결한 빈 객체 생성

> **JPA 비교:** JPA(Hibernate)는 Java 리플렉션의 `Constructor.newInstance()`로 엔티티를 생성한다.
> 이 메서드가 동작하려면 **인자 없는 생성자(기본 생성자)** 가 반드시 필요하다.

#### JPA: 기본 생성자 필수 (public 또는 protected)

```java
@Entity
public class User {
    protected User() {}  // 기본 생성자 — Hibernate가 리플렉션으로 호출

    public User(String name) {
        this.name = name;
    }
}
```

- `public` 또는 `protected` — Hibernate는 `setAccessible(true)`로 접근 제어를 우회하므로 `protected`도 동작
- `protected`를 쓰는 이유: 애플리케이션 코드에서 불완전한 엔티티를 직접 생성하는 것을 방지 (Hibernate만 사용하도록)
- `private`은 JPA 스펙에서 금지 — 일부 구현체에서 동작할 수 있지만 표준 위반

#### TypeORM: 생성자 제약 없음

```typescript
@Entity()
class User {
    private constructor() {} // private도 OK
    // 파라미터 있는 생성자만 있어도 OK
    // 생성자 없어도 OK (JS 기본 생성자)
}
```

TypeORM에서 private 생성자가 동작하는 이유:

1. **TypeScript `private`은 컴파일 타임에만 존재** — 트랜스파일 후 JS에서는 완전히 사라진다
2. **`Object.create()`는 생성자를 호출하지 않는다** — 프로토타입 체인만 연결한 빈 객체를 만든다

```typescript
// TypeScript 소스
class User {
    private constructor() {}
}

// 트랜스파일된 JavaScript — private이 사라짐
class User {
    constructor() {}
}
```

> **참고:** JavaScript의 `#` (진짜 private)과 TypeScript의 `private`은 다르다.
> `#`은 런타임에 실제로 접근이 차단되지만, TS `private`은 컴파일러만 체크한다.
> TypeORM 엔티티는 TS `private`을 사용하므로 런타임에 아무 제약이 없다.

#### 비교 요약

|                        | JPA (Hibernate)                        | TypeORM                                        |
| ---------------------- | -------------------------------------- | ---------------------------------------------- |
| **인스턴스 생성 방식** | `Constructor.newInstance()` (리플렉션) | `new target()` 또는 `Object.create(prototype)` |
| **기본 생성자**        | 필수 (public/protected)                | 불필요                                         |
| **protected 생성자**   | 동작 (`setAccessible(true)`)           | 동작 (TS protected = 컴파일 타임 전용)         |
| **private 생성자**     | JPA 스펙 위반                          | 동작 (TS private은 런타임에 사라짐)            |
| **접근 제어**          | 런타임 강제 (Java)                     | 컴파일 타임 전용 (TypeScript)                  |
| **final 클래스**       | 프록시 생성 불가 (lazy loading 제한)   | 해당 없음 (JS에 final 없음)                    |

---

### Step 3: EntityManager / Repository 사용

```typescript
// DataSource에 고정된 EntityManager 사용
const user = await dataSource.manager.findOne(User, { where: { id: 1 } })

// 또는 Repository 사용
const userRepo = dataSource.getRepository(User)
const user = await userRepo.findOne({ where: { id: 1 } })
```

`getRepository()`는 내부적으로 `dataSource.manager.getRepository()`를 호출한다 (`src/data-source/DataSource.ts:464-468`):

```typescript
getRepository(target) {
    return this.manager.getRepository(target)
}
```

---

## 핵심 차이: EntityManager의 수명과 역할

```
JPA:
  EntityManagerFactory (앱 1개) ──→ EntityManager (요청마다 새로 생성)
                                    └─ 영속성 컨텍스트 포함 (1차 캐시, 쓰기 지연, dirty checking)

TypeORM:
  DataSource (앱 1개) ──→ EntityManager (앱 1개, DataSource에 고정)
                           └─ 영속성 컨텍스트 없음 (상태 없는 파사드)
```

- JPA의 `EntityManager`는 **상태가 있다** — 영속성 컨텍스트를 가지며 요청마다 생성/소멸
- TypeORM의 `EntityManager`는 **상태가 없다** — 메서드만 제공하는 파사드, 앱 전체에서 1개 재사용

TypeORM의 `EntityManager`는 JPA의 `EntityManager`보다 Spring의 `JpaRepository`에 더 가깝다.
둘 다 stateless하고 앱 전체에서 재사용되며, 내부적으로 진짜 EntityManager(또는 DataSource)에 위임하기 때문이다.

---

## 대응 관계 요약

| JPA                                             | TypeORM                                                | 소스 위치                             |
| ----------------------------------------------- | ------------------------------------------------------ | ------------------------------------- |
| `persistence.xml`                               | `DataSourceOptions` (JS 객체)                          | 사용자 코드                           |
| `Persistence.createEntityManagerFactory()`      | `new DataSource(options)` + `initialize()`             | `src/data-source/DataSource.ts`       |
| `EntityManagerFactory`                          | `DataSource`                                           | `src/data-source/DataSource.ts`       |
| `SessionFactory` 빌드 (Hibernate)               | `buildMetadatas()`                                     | `src/data-source/DataSource.ts:734`   |
| `emf.createEntityManager()`                     | 해당 없음 (EntityManager가 DataSource에 고정)          | —                                     |
| `EntityManager` (요청마다, 상태 있음)           | `EntityManager` (앱 1개, 상태 없음)                    | `src/entity-manager/EntityManager.ts` |
| 영속성 컨텍스트                                 | 해당 없음                                              | —                                     |
| `@PersistenceContext EntityManager em` (Spring) | `dataSource.manager` 또는 `dataSource.getRepository()` | `src/data-source/DataSource.ts:464`   |

---

## 관련 소스 파일

| 파일                                          | 역할                                                          |
| --------------------------------------------- | ------------------------------------------------------------- |
| `src/data-source/DataSource.ts`               | 생성자, `initialize()`, `buildMetadatas()`, `getRepository()` |
| `src/data-source/DataSourceOptions.ts`        | 설정 옵션 타입 정의                                           |
| `src/connection/ConnectionMetadataBuilder.ts` | 엔티티/subscriber/migration 메타데이터 빌드                   |
| `src/entity-manager/EntityManager.ts`         | stateless 파사드 — `find()`, `save()` 등                      |
| `src/entity-manager/EntityManagerFactory.ts`  | EntityManager 인스턴스 생성                                   |
| `src/driver/DriverFactory.ts`                 | DB 타입에 맞는 Driver 구현체 생성                             |
