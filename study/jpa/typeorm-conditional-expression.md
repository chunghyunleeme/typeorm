# 조건식 (CASE, COALESCE, NULLIF)

## 개요

JPQL은 SQL과 유사한 조건식을 표준으로 제공한다.
TypeORM QueryBuilder는 이들에 대한 **전용 메서드가 없으며**, raw SQL 문자열로 작성해야 한다.

---

## 1. CASE 식

값에 따라 다른 결과를 반환하는 조건 분기.

### 기본 CASE vs 단순 CASE

```sql
-- 기본 CASE (searched case)
CASE WHEN 조건1 THEN 결과1 WHEN 조건2 THEN 결과2 ELSE 기본값 END

-- 단순 CASE (simple case)
CASE 대상 WHEN 값1 THEN 결과1 WHEN 값2 THEN 결과2 ELSE 기본값 END
```

### JPA (JPQL)

```java
// 기본 CASE
"SELECT " +
"  CASE WHEN e.salary >= 10000 THEN '고액' " +
"       WHEN e.salary >= 5000  THEN '중간' " +
"       ELSE '초급' END " +
"FROM Employee e"

// 단순 CASE
"SELECT " +
"  CASE e.department " +
"       WHEN 'IT'    THEN '기술' " +
"       WHEN 'HR'    THEN '인사' " +
"       ELSE '기타' END " +
"FROM Employee e"
```

### JPA (Criteria API)

```java
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<String> cq = cb.createQuery(String.class);
Root<Employee> e = cq.from(Employee.class);

// 기본 CASE — selectCase()
cq.select(
    cb.selectCase()
      .when(cb.ge(e.get("salary"), 10000), "고액")
      .when(cb.ge(e.get("salary"), 5000), "중간")
      .otherwise("초급")
);

// 단순 CASE — selectCase(expression)
cq.select(
    cb.selectCase(e.get("department"))
      .when("IT", "기술")
      .when("HR", "인사")
      .otherwise("기타")
);
```

### TypeORM QueryBuilder

전용 메서드 없음. `select()` / `addSelect()`에 raw SQL로 작성한다.

```typescript
// SELECT 절에서 CASE
dataSource
    .createQueryBuilder(Employee, "e")
    .select("e.name", "name")
    .addSelect(
        `CASE WHEN e.salary >= 10000 THEN '고액'
              WHEN e.salary >= 5000  THEN '중간'
              ELSE '초급' END`,
        "grade",
    )
    .getRawMany()

// WHERE 절에서 CASE
dataSource
    .createQueryBuilder(Employee, "e")
    .where(
        `CASE WHEN e.department = 'IT' THEN e.salary ELSE 0 END > :threshold`,
        { threshold: 5000 },
    )
    .getMany()

// ORDER BY에서 CASE (커스텀 정렬)
dataSource
    .createQueryBuilder(Employee, "e")
    .orderBy(`CASE e.department WHEN 'IT' THEN 1 WHEN 'HR' THEN 2 ELSE 3 END`)
    .getMany()
```

---

## 2. COALESCE

인자를 순서대로 검사하여 **첫 번째 NULL이 아닌 값**을 반환한다.

```sql
COALESCE(값1, 값2, ...) → 첫 번째 non-null 값
```

### JPA (JPQL)

```java
"SELECT COALESCE(e.nickname, e.name, '이름없음') FROM Employee e"
```

### JPA (Criteria API)

```java
CriteriaBuilder cb = em.getCriteriaBuilder();

// coalesce 체이닝
cb.coalesce()
  .value(e.get("nickname"))
  .value(e.get("name"))
  .value(cb.literal("이름없음"));

// 단순 2개 인자
cb.coalesce(e.get("nickname"), "이름없음");
```

### TypeORM QueryBuilder

전용 메서드 없음. raw SQL로 작성한다.

```typescript
// SELECT 절
dataSource
    .createQueryBuilder(Employee, "e")
    .select("COALESCE(e.nickname, e.name, '이름없음')", "displayName")
    .getRawMany()

// WHERE 절
dataSource
    .createQueryBuilder(Employee, "e")
    .where("COALESCE(e.nickname, e.name) = :name", { name: "Tom" })
    .getMany()
```

### TypeORM 내부 사용 사례

TypeORM은 내부적으로 COALESCE를 사용한다:

- **Generated Column**: `asExpression: "md5(coalesce(firstName,'0'))"` 형태로 생성 컬럼 정의
- **TreeRepository**: Materialized Path 트리 쿼리에서 경로 조합 시 사용

---

## 3. NULLIF

두 값이 **같으면 NULL**, 다르면 **첫 번째 값**을 반환한다.

```sql
NULLIF(값1, 값2) → 값1 = 값2이면 NULL, 아니면 값1
```

주로 **0으로 나누기 방지**에 사용한다.

### JPA (JPQL)

```java
// 0으로 나누기 방지
"SELECT e.totalSales / NULLIF(e.salesCount, 0) FROM Employee e"

// 빈 문자열을 NULL로 변환
"SELECT COALESCE(NULLIF(e.nickname, ''), e.name) FROM Employee e"
```

### JPA (Criteria API)

```java
CriteriaBuilder cb = em.getCriteriaBuilder();

// NULLIF
cb.nullif(e.get("salesCount"), 0);

// 0으로 나누기 방지
cb.quot(
    e.get("totalSales"),
    cb.nullif(e.get("salesCount"), 0)
);
```

### TypeORM QueryBuilder

전용 메서드 없음. raw SQL로 작성한다.

```typescript
// 0으로 나누기 방지
dataSource
    .createQueryBuilder(Employee, "e")
    .select("e.totalSales / NULLIF(e.salesCount, 0)", "avgSale")
    .getRawMany()

// 빈 문자열을 NULL로 변환하여 COALESCE와 조합
dataSource
    .createQueryBuilder(Employee, "e")
    .select("COALESCE(NULLIF(e.nickname, ''), e.name)", "displayName")
    .getRawMany()
```

### TypeORM 내부 사용 사례

TreeRepository의 Materialized Path 쿼리에서 사용한다:

```typescript
// src/repository/TreeRepository.ts
;`NULLIF(CONCAT(${subQuery.getQuery()}, '%'), '%')`
```

서브쿼리 결과가 빈 문자열일 때 `LIKE '%'`로 전체 매칭되는 것을 방지하기 위해
NULLIF로 `'%'`와 같으면 NULL을 반환하게 한다.

---

## CASE + COALESCE + NULLIF 조합 패턴

실무에서는 세 조건식을 조합하여 사용하는 경우가 많다.

### JPA (JPQL)

```java
// 닉네임이 비어있으면 이름 사용, 급여 구간별 등급 표시
"SELECT " +
"  COALESCE(NULLIF(e.nickname, ''), e.name), " +
"  CASE WHEN e.salary / NULLIF(e.workMonths, 0) >= 500 THEN 'A' " +
"       ELSE 'B' END " +
"FROM Employee e"
```

### TypeORM QueryBuilder

```typescript
dataSource
    .createQueryBuilder(Employee, "e")
    .select("COALESCE(NULLIF(e.nickname, ''), e.name)", "displayName")
    .addSelect(
        `CASE WHEN e.salary / NULLIF(e.workMonths, 0) >= 500 THEN 'A'
              ELSE 'B' END`,
        "grade",
    )
    .getRawMany()
```

---

## 핵심 비교 요약

| 조건식       | JPA (JPQL)                      | JPA (Criteria API)                   | TypeORM QueryBuilder |
| ------------ | ------------------------------- | ------------------------------------ | -------------------- |
| **CASE**     | `CASE WHEN ... END`             | `cb.selectCase().when().otherwise()` | raw SQL 문자열       |
| **COALESCE** | `COALESCE(a, b, c)`             | `cb.coalesce().value(a).value(b)`    | raw SQL 문자열       |
| **NULLIF**   | `NULLIF(a, b)`                  | `cb.nullif(a, b)`                    | raw SQL 문자열       |
| 타입 안전성  | JPQL 런타임 파싱                | 컴파일 타임 체크                     | 없음 (순수 문자열)   |
| 사용 위치    | SELECT, WHERE, HAVING, ORDER BY | 동일                                 | 동일                 |

### 핵심 차이

1. **JPA Criteria API는 전용 빌더 메서드**(`selectCase()`, `coalesce()`, `nullif()`)를 제공한다.
2. **TypeORM은 전용 메서드가 없다** — `select()`, `addSelect()`, `where()`에 raw SQL로 작성한다.
3. JPA는 Criteria API로 **타입 안전성과 IDE 자동완성**을 얻을 수 있지만, TypeORM은 문자열이므로 **오타나 문법 오류를 런타임에서야 발견**한다.
4. 실질적으로 TypeORM에서 이 조건식들은 SQL 함수를 그대로 쓰는 것과 다르지 않다 — ORM이 추상화하지 않는 영역이다.

---

## 설계 개선 이슈

위 분석에서 도출된 TypeORM 개선사항을 upstream에 이슈로 등록했다.

| 이슈                                                      | 개선 내용                                                 | 현재 상태                                            |
| --------------------------------------------------------- | --------------------------------------------------------- | ---------------------------------------------------- |
| [#12069](https://github.com/typeorm/typeorm/issues/12069) | CASE 식 빌더 메서드 추가 (`qb.case().when().otherwise()`) | 전용 메서드 없이 raw SQL 문자열만 가능               |
| [#12070](https://github.com/typeorm/typeorm/issues/12070) | `coalesce()` 헬퍼 메서드 추가                             | 내부적으로 사용하지만 사용자 API 미제공              |
| [#12071](https://github.com/typeorm/typeorm/issues/12071) | `nullif()` 헬퍼 메서드 추가                               | TreeRepository에서 내부 사용하지만 사용자 API 미제공 |

공통 개선 방향: JPA Criteria API처럼 조건식을 **전용 빌더/헬퍼 메서드**로 제공하여
raw SQL 문자열 의존도를 줄이고, 파라미터 바인딩과 IDE 자동완성을 지원한다.
