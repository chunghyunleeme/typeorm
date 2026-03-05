# 의존성 스택 (Spring Boot JPA vs TypeORM)

## 개요

Spring Boot에서 JPA를 사용하면 여러 계층의 라이브러리가 **자동으로 구성**된다.
TypeORM은 이 모든 계층을 **하나의 npm 패키지**가 담당하며, DB 드라이버만 별도로 설치한다.

---

## Spring Boot JPA 의존성 스택

```
spring-boot-starter-data-jpa
├── spring-data-jpa          ← Repository 추상화 (JpaRepository, @Query, 메서드 이름 쿼리)
├── hibernate-core            ← JPA 구현체 (엔티티 매핑, JPQL, 영속성 컨텍스트, 캐시)
│   └── jakarta.persistence-api  ← JPA 표준 스펙 인터페이스
├── spring-boot-starter-jdbc  ← JDBC 기반 인프라
│   ├── HikariCP              ← 커넥션 풀 (기본 내장)
│   └── spring-jdbc           ← JdbcTemplate, 트랜잭션 관리
└── DB Driver                 ← JDBC 드라이버 (postgresql, mysql-connector-java 등)
```

### 각 계층의 역할

| 계층                  | 라이브러리              | 역할                                                            |
| --------------------- | ----------------------- | --------------------------------------------------------------- |
| **Repository 추상화** | spring-data-jpa         | `JpaRepository`, `findBy...` 자동 쿼리, `@Query`, 페이징        |
| **ORM 구현체**        | hibernate-core          | 엔티티 매핑, JPQL 파싱, 영속성 컨텍스트, 1차/2차 캐시, DDL 생성 |
| **JPA 표준 스펙**     | jakarta.persistence-api | `@Entity`, `@Id`, `EntityManager` 등 표준 인터페이스            |
| **커넥션 풀**         | HikariCP                | 커넥션 풀 관리, 성능 최적화 (Spring Boot 기본)                  |
| **JDBC 지원**         | spring-jdbc             | `JdbcTemplate`, `DataSource` 추상화, 트랜잭션 매니저            |
| **DB 드라이버**       | postgresql 등           | JDBC 프로토콜로 DB와 통신                                       |

### 핵심: 표준과 구현의 분리

```java
// JPA 표준 인터페이스 (jakarta.persistence)
@Entity                     // 표준
@Id                         // 표준
EntityManager em;           // 표준

// Hibernate 구현 (org.hibernate)
Session session;            // Hibernate 전용
@Cache                      // Hibernate 전용 2차 캐시

// Spring Data JPA 추상화
JpaRepository<User, Long>   // Spring 전용
```

JPA 표준을 지키면 Hibernate → EclipseLink 등 **구현체를 교체**할 수 있다.
실무에서 교체하는 경우는 거의 없지만, 표준 스펙이 있다는 것 자체가 API 설계의 안정성을 보장한다.

---

## TypeORM 의존성 스택

```
typeorm                       ← ORM 전체 (엔티티 매핑 + QueryBuilder + Repository + 커넥션 풀 관리)
└── DB Driver (직접 설치)     ← pg, mysql2, better-sqlite3, mssql 등
```

### 단일 패키지가 모든 계층을 담당

| Spring Boot JPA 계층    | TypeORM 대응                                                 | 위치                                        |
| ----------------------- | ------------------------------------------------------------ | ------------------------------------------- |
| spring-data-jpa         | `Repository`, `Repository.extend()`                          | `src/repository/`                           |
| hibernate-core          | `EntityManager`, `QueryBuilder`, 메타데이터                  | `src/entity-manager/`, `src/query-builder/` |
| jakarta.persistence-api | 데코레이터 (`@Entity`, `@Column`, `@PrimaryGeneratedColumn`) | `src/decorator/`                            |
| HikariCP                | DB 드라이버의 내장 풀 위임                                   | `src/driver/`                               |
| spring-jdbc             | `Driver`, `QueryRunner`                                      | `src/driver/`, `src/query-runner/`          |
| DB Driver               | `pg`, `mysql2`, `mssql` 등 (별도 설치)                       | `node_modules/`                             |

---

## 커넥션 풀 비교

### HikariCP (Spring Boot 기본)

```yaml
# application.yml
spring:
    datasource:
        hikari:
            maximum-pool-size: 10 # 최대 커넥션 수
            minimum-idle: 5 # 최소 유휴 커넥션
            connection-timeout: 30000 # 커넥션 획득 대기 (ms)
            idle-timeout: 600000 # 유휴 커넥션 제거 시간
            max-lifetime: 1800000 # 커넥션 최대 수명
            pool-name: MyHikariPool
```

HikariCP는 **독립적인 커넥션 풀 라이브러리**다.
JDBC DataSource를 감싸서 풀링을 제공하며, Spring Boot가 기본으로 선택한다.

### TypeORM — DB 드라이버 내장 풀에 위임

TypeORM은 자체 커넥션 풀이 없다. **DB 드라이버의 내장 풀**에 위임한다.

```typescript
// DataSource 설정
const dataSource = new DataSource({
    type: "postgres",
    host: "localhost",
    port: 5432,
    poolSize: 10, // pg.Pool의 max 옵션으로 매핑
    extra: {
        // 드라이버에 직접 전달되는 옵션
        connectionTimeoutMillis: 30000,
        idleTimeoutMillis: 600000,
    },
})
```

드라이버별 풀 매핑:

| TypeORM 옵션 | PostgreSQL (`pg.Pool`) | MySQL (`mysql2.createPool`) | MSSQL (`mssql.ConnectionPool`) |
| ------------ | ---------------------- | --------------------------- | ------------------------------ |
| `poolSize`   | `max`                  | `connectionLimit`           | `pool.max` (별도 객체)         |
| `extra`      | 스프레드 병합          | 스프레드 병합               | 스프레드 병합                  |

```typescript
// TypeORM 내부 — PostgresDriver.ts
// poolSize를 pg.Pool의 max로 변환
const pool = new this.postgres.Pool({
    ...connectionOptions,
    max: options.poolSize, // TypeORM poolSize → pg max
    ...options.extra, // 드라이버 직접 옵션
})
```

### 핵심 차이

| 항목                   | HikariCP                                | TypeORM                    |
| ---------------------- | --------------------------------------- | -------------------------- |
| **풀 구현**            | 독립 라이브러리 (교체 가능)             | DB 드라이버 내장 풀 위임   |
| **설정 통일성**        | 모든 DB에 동일한 설정                   | DB별로 `extra` 옵션이 다름 |
| **커넥션 유효성 검사** | `connectionTestQuery` / `keepaliveTime` | 드라이버 자체 기능에 의존  |
| **메트릭/모니터링**    | MBean 자동 등록, Micrometer 연동        | 별도 지원 없음             |
| **교체 가능성**        | Tomcat DBCP, Commons DBCP2 등 교체 가능 | 드라이버에 종속            |

---

## Replication (읽기/쓰기 분리)

### Spring Boot

```yaml
# 별도 DataSource 빈 구성 필요 (자동 구성 아님)
# AbstractRoutingDataSource로 Master/Slave 라우팅 구현
```

### TypeORM — 내장 지원

```typescript
const dataSource = new DataSource({
    type: "mysql",
    replication: {
        master: {
            host: "master.db.com",
            port: 3306,
            username: "root",
            password: "password",
            database: "mydb",
        },
        slaves: [
            {
                host: "slave1.db.com",
                port: 3306,
                username: "root",
                password: "password",
                database: "mydb",
            },
            {
                host: "slave2.db.com",
                port: 3306,
                username: "root",
                password: "password",
                database: "mydb",
            },
        ],
    },
})
```

TypeORM은 replication을 **내장 지원**한다.
MySQL은 `createPoolCluster()`로 MASTER/SLAVE 노드를 관리하고,
PostgreSQL은 별도의 master/slave 풀을 생성한다.

---

## 전체 아키텍처 비교

### Spring Boot JPA

```
Application Code
    ↓
Spring Data JPA (Repository 인터페이스)
    ↓
Hibernate (JPA 구현체)
    ↓
JPA 표준 인터페이스 (jakarta.persistence)
    ↓
HikariCP (커넥션 풀)
    ↓
JDBC Driver (postgresql 등)
    ↓
Database
```

- 각 계층이 **독립적으로 교체 가능** (Hibernate → EclipseLink, HikariCP → DBCP2)
- 표준 스펙(JPA, JDBC)이 계층 간 **계약**을 정의
- Spring Boot가 Auto Configuration으로 **자동 조립**

### TypeORM

```
Application Code
    ↓
TypeORM (Repository + EntityManager + QueryBuilder + Driver 추상화)
    ↓
DB Driver (pg, mysql2, mssql 등)
    ↓
Database
```

- **단일 패키지**가 ORM 전체 기능을 제공
- 표준 스펙 없이 TypeORM API가 곧 계약
- DB 드라이버만 교체 가능 (ORM 자체는 교체 불가)

---

## 핵심 비교 요약

| 항목            | Spring Boot JPA                                                     | TypeORM                        |
| --------------- | ------------------------------------------------------------------- | ------------------------------ |
| **패키지 수**   | 6+ (starter, spring-data, hibernate, HikariCP, spring-jdbc, driver) | 2 (typeorm + driver)           |
| **표준 스펙**   | JPA (jakarta.persistence), JDBC                                     | 없음                           |
| **ORM 교체**    | Hibernate ↔ EclipseLink 가능                                        | 불가 (TypeORM = 유일한 구현)   |
| **커넥션 풀**   | HikariCP (독립, 교체 가능)                                          | 드라이버 내장 풀 위임          |
| **Replication** | 직접 구현 (AbstractRoutingDataSource)                               | 내장 지원 (`replication` 옵션) |
| **자동 구성**   | Spring Boot Auto Configuration                                      | `DataSource` 수동 생성         |
| **설정 방식**   | `application.yml` 통합                                              | `DataSource` 생성자 옵션       |

### 핵심 차이

1. **Spring Boot JPA는 계층화된 아키텍처**다. 표준 스펙(JPA, JDBC)으로 각 계층이 분리되어 교체 가능하다.
   TypeORM은 **모놀리식 단일 패키지**로, 간단하지만 교체 유연성이 없다.

2. **커넥션 풀 전략이 근본적으로 다르다.** Spring Boot는 HikariCP라는 전문 풀 라이브러리를 사용하고,
   TypeORM은 각 DB 드라이버의 내장 풀에 위임한다. HikariCP 수준의 세밀한 풀 관리(메트릭, 유효성 검사 등)는 TypeORM에서 어렵다.

3. **Spring Boot의 Auto Configuration은 관례 기반 자동 조립**을 제공한다.
   의존성만 추가하면 DataSource, EntityManagerFactory, TransactionManager가 자동 생성된다.
   TypeORM은 `new DataSource({...})`로 수동 구성해야 한다.
