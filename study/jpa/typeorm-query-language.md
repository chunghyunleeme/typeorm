# JPQL vs TypeORM 쿼리 방식

## 개요

JPQL(Java Persistence Query Language)은 **엔티티 객체를 대상으로 하는 쿼리 언어**다.
SQL이 테이블을 대상으로 하는 것과 달리, JPQL은 엔티티 클래스와 필드명을 사용한다.

```java
// SQL  — 테이블명, 컬럼명
SELECT u.user_name FROM users u WHERE u.user_id = 1

// JPQL — 엔티티명, 필드명
SELECT u.name FROM User u WHERE u.id = 1
```

TypeORM에는 JPQL이라는 단일 쿼리 언어가 없다.
대신 **Find API**, **QueryBuilder**, **Raw SQL** 3가지 방식을 제공한다.

---

## 1. Find API (가장 간단)

```java
// JPA
em.find(User.class, 1L);

List<User> users = em.createQuery(
    "SELECT u FROM User u WHERE u.age > 20", User.class
).getResultList();
```

```typescript
// TypeORM Find API
userRepo.findOne({ where: { id: 1 } })

userRepo.find({
    where: { age: MoreThan(20) },
    relations: ["posts"],
    order: { name: "ASC" },
})
```

JPQL 없이도 간단한 조회가 가능하다.
JPA의 Spring Data JPA `findBy...` 메서드와 비슷한 위치.

---

## 2. QueryBuilder (JPQL에 가장 가까움)

### 비교 예시

```java
// JPQL
String jpql = "SELECT u FROM User u " +
              "LEFT JOIN FETCH u.posts p " +
              "WHERE u.age > :age " +
              "ORDER BY u.name";
em.createQuery(jpql, User.class)
  .setParameter("age", 20)
  .getResultList();
```

```typescript
// TypeORM QueryBuilder
dataSource
    .createQueryBuilder(User, "u")
    .leftJoinAndSelect("u.posts", "p")
    .where("u.age > :age", { age: 20 })
    .orderBy("u.name")
    .getMany()
```

### 절별 대응

| JPQL                        | QueryBuilder                          |
| --------------------------- | ------------------------------------- |
| `SELECT u FROM User u`      | `.createQueryBuilder(User, "u")`      |
| `LEFT JOIN FETCH u.posts p` | `.leftJoinAndSelect("u.posts", "p")`  |
| `WHERE u.age > :age`        | `.where("u.age > :age", { age: 20 })` |
| `ORDER BY u.name`           | `.orderBy("u.name")`                  |
| `GROUP BY u.department`     | `.groupBy("u.department")`            |
| `HAVING COUNT(p) > 5`       | `.having("COUNT(p) > 5")`             |
| `.setParameter("age", 20)`  | `.where()` 두 번째 인자로 통합        |
| `.getResultList()`          | `.getMany()`                          |
| `.getSingleResult()`        | `.getOne()`                           |

JPQL은 **문자열**로 쿼리를 작성하고, QueryBuilder는 **메서드 체이닝**으로 쿼리를 조립한다.
하지만 둘 다 **엔티티 필드명(DB 컬럼명 아님)** 을 사용한다는 점은 같다.

### 서브쿼리

```java
// JPQL
"SELECT u FROM User u WHERE u.age > (SELECT AVG(u2.age) FROM User u2)"
```

```typescript
// TypeORM QueryBuilder
dataSource
    .createQueryBuilder(User, "u")
    .where(
        "u.age > " +
            dataSource
                .createQueryBuilder()
                .subQuery()
                .select("AVG(u2.age)")
                .from(User, "u2")
                .getQuery(),
    )
    .getMany()
```

### INSERT / UPDATE / DELETE

```java
// JPQL bulk update
em.createQuery("UPDATE User u SET u.name = :name WHERE u.id = :id")
  .setParameter("name", "Tom")
  .setParameter("id", 1)
  .executeUpdate();
```

```typescript
// TypeORM QueryBuilder
dataSource
    .createQueryBuilder()
    .update(User)
    .set({ name: "Tom" })
    .where("id = :id", { id: 1 })
    .execute()
```

---

## 3. Raw SQL (네이티브 쿼리)

```java
// JPA Native Query
em.createNativeQuery("SELECT * FROM users WHERE age > ?", User.class)
  .setParameter(1, 20)
  .getResultList();
```

```typescript
// TypeORM Raw Query
manager.query("SELECT * FROM users WHERE age > $1", [20])
```

Raw SQL은 ORM을 우회하여 직접 SQL을 실행한다.
TypeORM의 `query()`는 엔티티로 변환하지 않고 plain object를 반환한다.

---

## 핵심 차이

|                    | JPQL                            | TypeORM QueryBuilder                              |
| ------------------ | ------------------------------- | ------------------------------------------------- |
| **형태**           | 문자열 (SQL과 비슷한 별도 언어) | 메서드 체이닝 (TypeScript 코드)                   |
| **컴파일 시 검증** | 불가 (런타임 파싱)              | 부분적 가능 (메서드명은 체크, 문자열 인자는 불가) |
| **엔티티 기반**    | `SELECT u FROM User u`          | `.createQueryBuilder(User, "u")`                  |
| **서브쿼리**       | 문자열 내 중첩                  | `.subQuery()` 메서드                              |
| **flush 트리거**   | JPQL 실행 전 자동 flush         | 해당 없음 (영속성 컨텍스트 없음)                  |
| **Criteria API**   | JPA 표준 빌더 패턴 (별도 존재)  | QueryBuilder가 이 역할 겸함                       |

### flush 트리거 차이

JPA에서 JPQL을 실행하면 **자동으로 flush가 먼저 일어난다**.
영속성 컨텍스트에 쌓인 변경사항을 DB에 반영한 뒤 쿼리해야 정합성이 맞기 때문이다.

```java
em.persist(newUser);    // INSERT 아직 안 됨 (쓰기 지연)

// JPQL 실행 → 자동 flush → INSERT 실행 → 그 다음 SELECT 실행
List<User> users = em.createQuery("SELECT u FROM User u").getResultList();
// newUser가 결과에 포함됨
```

TypeORM은 영속성 컨텍스트가 없으므로 이런 자동 flush 개념 자체가 없다.

---

## 3가지 방식의 선택 기준

| 상황                       | JPA                         | TypeORM                           |
| -------------------------- | --------------------------- | --------------------------------- |
| 단순 PK 조회               | `em.find()`                 | `repo.findOne()`                  |
| 간단한 조건 조회           | Spring Data JPA `findBy...` | Find API (`find({ where: ... })`) |
| 복잡한 조건, JOIN          | JPQL 또는 Criteria API      | QueryBuilder                      |
| DB 종속적 SQL, 성능 최적화 | Native Query                | `manager.query()`                 |

JPQL은 **독립된 쿼리 언어**이고, TypeORM QueryBuilder는 **빌더 패턴 API**이다.
접근 방식은 다르지만, 둘 다 "테이블이 아닌 엔티티를 대상으로 쿼리한다"는 ORM의 목적은 같다.
