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
    entities: [User, Post],    // 엔티티 클래스 직접 전달
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

| JPA | TypeORM | 소스 위치 |
|---|---|---|
| `persistence.xml` | `DataSourceOptions` (JS 객체) | 사용자 코드 |
| `Persistence.createEntityManagerFactory()` | `new DataSource(options)` + `initialize()` | `src/data-source/DataSource.ts` |
| `EntityManagerFactory` | `DataSource` | `src/data-source/DataSource.ts` |
| `SessionFactory` 빌드 (Hibernate) | `buildMetadatas()` | `src/data-source/DataSource.ts:734` |
| `emf.createEntityManager()` | 해당 없음 (EntityManager가 DataSource에 고정) | — |
| `EntityManager` (요청마다, 상태 있음) | `EntityManager` (앱 1개, 상태 없음) | `src/entity-manager/EntityManager.ts` |
| 영속성 컨텍스트 | 해당 없음 | — |
| `@PersistenceContext EntityManager em` (Spring) | `dataSource.manager` 또는 `dataSource.getRepository()` | `src/data-source/DataSource.ts:464` |

---

## 관련 소스 파일

| 파일 | 역할 |
|---|---|
| `src/data-source/DataSource.ts` | 생성자, `initialize()`, `buildMetadatas()`, `getRepository()` |
| `src/data-source/DataSourceOptions.ts` | 설정 옵션 타입 정의 |
| `src/connection/ConnectionMetadataBuilder.ts` | 엔티티/subscriber/migration 메타데이터 빌드 |
| `src/entity-manager/EntityManager.ts` | stateless 파사드 — `find()`, `save()` 등 |
| `src/entity-manager/EntityManagerFactory.ts` | EntityManager 인스턴스 생성 |
| `src/driver/DriverFactory.ts` | DB 타입에 맞는 Driver 구현체 생성 |