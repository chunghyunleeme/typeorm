# TypeORM 아키텍처 계층 구조 (JPA 비교)

## 개요

JPA는 애플리케이션과 JDBC 사이에서 동작하며, Java 진영은 각 계층이 표준 스펙으로 분리되어 있다.
TypeORM은 Node.js 환경에서 ORM + Driver 추상화를 모두 자체적으로 갖고 있으며, JDBC 같은 공통 표준 없이 DB별 npm 패키지를 직접 감싼다.

---

## 계층 비교

```
  JPA (Java)                            TypeORM (Node.js)
  계층 = 스펙 경계                       계층 = 역할 분담

  Application Code                      Application Code
       │                                     │
       ▼                                     ▼
  ┌──────────────────┐ 인터페이스     ┌──────────────────┐ 구체 클래스
  │ JPA API (표준 스펙)│              │ Repository        │ 진입점
  │ EntityManager     │              │ EntityManager     │ (findOne, save 등)
  └────────┬─────────┘              │ QueryBuilder      │ SQL 조립
           │ 구현                    └────────┬─────────┘
           ▼                                  │ 위임
  ┌──────────────────┐ 구현체         ┌───────┴──────────┐ 유일한 인터페이스
  │ Hibernate         │              │ Driver 인터페이스   │
  │ 프록시, SQL, 캐시  │              │   ├ PostgresDriver │ DB별 구현체
  └────────┬─────────┘              │   ├ MysqlDriver    │
           │                         │   └ SqliteDriver   │
           ▼                         └────────┬──────────┘
  ┌──────────────────┐ 인터페이스              │ 호출
  │ JDBC (표준 스펙)   │                        ▼
  │ 표준 DB 접속 API  │              ┌──────────────────┐ 외부 패키지
  └────────┬─────────┘              │ npm 패키지         │
           │                         │ pg, mysql2,       │
           ▼                         │ better-sqlite3    │
       Database                      └────────┬─────────┘
                                              │
                                              ▼
                                          Database
```

JPA는 계층마다 **스펙(인터페이스) / 구현체** 경계가 있다.
TypeORM은 스펙/구현 분리가 **Driver에만** 존재하고, 나머지는 전부 구체 클래스의 **역할 분담**이다.

---

## 핵심 차이: 스펙/구현 분리 vs 역할 분담

### Java 진영 — 계층 = 스펙 경계 (인터페이스/구현체 분리)

- **JPA** — ORM 표준 스펙 (인터페이스)
- **Hibernate** — JPA 구현체 (교체 가능: EclipseLink, OpenJPA 등)
- **JDBC** — DB 접속 표준 스펙 (인터페이스)
- **JDBC Driver** — DB별 구현체 (`postgresql-42.x.jar` 등)

`EntityManager`는 **인터페이스**이고, Hibernate가 그 **구현체**를 제공한다.
표준이 있으므로 각 계층을 독립적으로 교체할 수 있다.

### Node.js 진영 — 계층 = 역할 분담 (구체 클래스의 위임 관계)

- **Repository, EntityManager, QueryBuilder** — 전부 **구체 클래스**. 인터페이스가 아니다.
- **Driver** — TypeORM에서 **유일한 인터페이스/구현체 패턴**. `Driver` 인터페이스를 `PostgresDriver`, `MysqlDriver` 등이 구현한다.
- **npm 패키지** — Driver 구현체가 감싸는 외부 라이브러리.

Repository → EntityManager → QueryBuilder는 **위임(delegation)** 관계이지, 스펙/구현 관계가 아니다.
ORM을 바꾸려면(TypeORM → Prisma 등) 애플리케이션 코드를 전면 수정해야 한다.

```
JPA:      EntityManager (인터페이스) ←── Hibernate가 구현
TypeORM:  Repository (구체 클래스) → EntityManager (구체 클래스) → QueryBuilder (구체 클래스) → Driver (인터페이스)
                                        위임 ────────────────────────────→                        ↑ 유일한 스펙/구현 분리
```

---

## 계층별 매핑

| JPA 계층                | 역할                        | TypeORM 대응                    | 소스 위치                                |
| ----------------------- | --------------------------- | ------------------------------- | ---------------------------------------- |
| JPA API (EntityManager) | 엔티티 CRUD, 쿼리           | Repository / EntityManager      | `src/repository/`, `src/entity-manager/` |
| Hibernate (구현체)      | 프록시 생성, SQL 생성, 캐시 | QueryBuilder, RelationLoader 등 | `src/query-builder/`                     |
| JDBC                    | 표준 DB 접속 인터페이스     | Driver 인터페이스               | `src/driver/Driver.ts`                   |
| JDBC Driver (jar)       | DB별 네이티브 구현          | npm 패키지 (`pg`, `mysql2` 등)  | `node_modules/pg` 등                     |

---

## TypeORM의 Driver 계층

### Driver 인터페이스

`src/driver/Driver.ts`:

```typescript
/**
 * Driver organizes TypeORM communication with specific database management system.
 */
export interface Driver {
    // 커넥션 옵션, 쿼리 실행, 스키마 빌드 등
}
```

JPA의 JDBC 역할에 해당하지만, 표준 스펙이 아니라 TypeORM 내부 인터페이스이다.

### 지원 DB별 Driver 구현

```
src/driver/
  ├── postgres/        PostgresDriver    → require("pg")
  ├── mysql/           MysqlDriver       → require("mysql2")
  ├── sqlite/          SqliteDriver      → require("sqlite3")
  ├── better-sqlite3/  BetterSqlite3Driver → require("better-sqlite3")
  ├── oracle/          OracleDriver      → require("oracledb")
  ├── sqlserver/       SqlServerDriver   → require("mssql")
  ├── cockroachdb/     CockroachDriver   → require("pg")
  ├── mongodb/         MongoDriver       → require("mongodb")
  ├── spanner/         SpannerDriver     → require("@google-cloud/spanner")
  └── ...
```

### 실제 코드: PostgreSQL Driver가 npm 패키지를 사용하는 방식

`src/driver/postgres/PostgresDriver.ts`:

```typescript
// npm 패키지 "pg"를 로드
this.postgres = options.driver || PlatformTools.load("pg")

// "pg" 패키지의 Pool을 직접 생성
const pool = new this.postgres.Pool(connectionOptions)
```

JPA에서 `DriverManager.getConnection(url)`으로 JDBC Driver를 통해 커넥션을 얻는 것처럼,
TypeORM은 Driver 구현체가 npm 패키지의 Pool/Connection 객체를 직접 생성하여 DB와 통신한다.

---

## 요약

```
Java:    App  →  JPA (표준)  →  Hibernate (구현체)  →  JDBC (표준)  →  JDBC Driver (jar)  →  DB
Node.js: App  →  TypeORM (Repository/EntityManager/QueryBuilder/Driver 전부 내장)  →  npm pkg  →  DB
```

Java는 계층마다 표준 스펙이 있어서 각 계층을 독립적으로 교체 가능하지만,
TypeORM은 하나의 라이브러리가 모든 계층을 담당하므로 교체 단위가 "TypeORM 전체"가 된다.
