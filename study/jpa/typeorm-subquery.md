# 서브쿼리 (Subquery)

## 개요

서브쿼리는 SQL 문 안에 포함된 또 다른 SELECT 문이다.
JPA(JPQL)는 서브쿼리를 제한적으로 지원하고, TypeORM QueryBuilder는 문자열 조합 방식으로 더 넓은 범위를 지원한다.

---

## JPA 서브쿼리 사용 가능 위치

| 위치          | JPQL (JPA 표준) | Hibernate (확장) | TypeORM QueryBuilder |
| ------------- | --------------- | ---------------- | -------------------- |
| **WHERE 절**  | O               | O                | O                    |
| **HAVING 절** | O               | O                | O                    |
| **SELECT 절** | X               | O (스칼라)       | O (`.addSelect()`)   |
| **FROM 절**   | X               | X                | O (`.from()`)        |

JPQL 표준은 **WHERE, HAVING 절에서만** 서브쿼리를 허용한다.
Hibernate는 SELECT 절의 스칼라 서브쿼리를 추가로 지원한다.
FROM 절 서브쿼리(인라인 뷰)는 JPQL에서 불가능하다 — **JOIN으로 풀 수 있으면 조인으로 해결**하는 것이 원칙이다.

---

## TypeORM `subQuery()` 내부 동작

TypeORM에서 서브쿼리를 만드는 핵심 메서드다.

```typescript
// src/query-builder/SelectQueryBuilder.ts
subQuery(): SelectQueryBuilder<any> {
    const qb = this.createQueryBuilder()
    qb.expressionMap.subQuery = true   // 서브쿼리 플래그 ON
    qb.parentQueryBuilder = this       // 부모 참조 저장
    return qb
}
```

하는 일은 3가지다:

1. **새 `SelectQueryBuilder` 인스턴스**를 생성한다
2. `expressionMap.subQuery = true` — 서브쿼리 플래그를 켠다
3. `parentQueryBuilder = this` — 부모 QueryBuilder 참조를 저장한다

서브쿼리 플래그가 `true`이면 `getQuery()` 호출 시 SQL을 **괄호로 감싸서** 반환한다.

```typescript
// src/query-builder/SelectQueryBuilder.ts:96
if (this.expressionMap.subQuery) sql = "(" + sql + ")"
```

따라서 `.subQuery().select(...).from(...).getQuery()`를 호출하면
`(SELECT ... FROM ...)` 형태의 문자열이 반환되고,
이를 메인 쿼리의 `.where()`에 문자열로 연결하는 방식이다.

JPA Criteria API의 `cq.subquery(Integer.class)`에 대응하지만,
**타입 정보 없이 순수 SQL 문자열만 생성**한다는 차이가 있다.

---

## 1. `[NOT] EXISTS (subquery)`

부서를 관리하는 직원이 **존재하는지** 확인.

### JPA (JPQL)

```java
// EXISTS
"SELECT e FROM Employee e WHERE EXISTS " +
"(SELECT 1 FROM Department d WHERE d.managerId = e.id)"

// NOT EXISTS
"SELECT e FROM Employee e WHERE NOT EXISTS " +
"(SELECT 1 FROM Department d WHERE d.managerId = e.id)"
```

### JPA (Criteria API)

```java
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<Employee> cq = cb.createQuery(Employee.class);
Root<Employee> e = cq.from(Employee.class);

// 서브쿼리 생성
Subquery<Integer> sub = cq.subquery(Integer.class);
Root<Department> d = sub.from(Department.class);
sub.select(cb.literal(1))
   .where(cb.equal(d.get("managerId"), e.get("id")));

// EXISTS
cq.where(cb.exists(sub));

// NOT EXISTS
cq.where(cb.not(cb.exists(sub)));
```

### TypeORM QueryBuilder

```typescript
// EXISTS — whereExists 메서드 제공
const subQuery = dataSource
    .createQueryBuilder(Department, "d")
    .where("d.managerId = e.id")

dataSource
    .createQueryBuilder(Employee, "e")
    .whereExists(subQuery) // EXISTS
    // .andWhereExists(subQuery)  — AND EXISTS
    // .orWhereExists(subQuery)   — OR EXISTS
    .getMany()
```

### TypeORM 내부 동작

`whereExists()`는 내부적으로 `getExistsCondition()`을 호출한다.

```typescript
// src/query-builder/QueryBuilder.ts
protected getExistsCondition(subQuery: any): [string, any[]] {
    const query = subQuery
        .clone()
        .orderBy()       // ORDER BY 제거
        .groupBy()       // GROUP BY 제거
        .offset(undefined)
        .limit(undefined)
        .select("1")     // SELECT 1로 최적화
        .setOption("disable-global-order")

    return [`EXISTS (${query.getQuery()})`, query.getParameters()]
}
```

서브쿼리를 클론한 뒤 **ORDER BY, GROUP BY, LIMIT 등을 제거**하고 `SELECT 1`로 치환한다.
EXISTS는 행의 존재 여부만 확인하므로 이런 최적화가 가능하다.

### 한계: NOT EXISTS 미지원

TypeORM은 `whereExists()`는 있지만 `whereNotExists()`는 **없다**.
NOT EXISTS가 필요하면 문자열로 직접 작성해야 한다.

```typescript
const subQuery = dataSource
    .createQueryBuilder()
    .subQuery()
    .select("1")
    .from(Department, "d")
    .where("d.managerId = e.id")
    .getQuery()

dataSource
    .createQueryBuilder(Employee, "e")
    .where(`NOT EXISTS ${subQuery}`)
    .getMany()
```

---

## 2. `{ALL | ANY | SOME} (subquery)`

집계 서브쿼리 결과와 비교할 때 사용한다.

| 키워드 | 의미                                         |
| ------ | -------------------------------------------- |
| `ALL`  | 서브쿼리의 **모든** 결과와 조건을 만족       |
| `ANY`  | 서브쿼리의 **하나라도** 조건을 만족 (= SOME) |
| `SOME` | ANY와 동일                                   |

### JPA (JPQL)

```java
// ALL — 모든 부서의 평균 급여보다 높은 직원
"SELECT e FROM Employee e WHERE e.salary > ALL " +
"(SELECT AVG(e2.salary) FROM Employee e2 GROUP BY e2.departmentId)"

// ANY — 어느 하나의 부서 평균 급여보다 높은 직원
"SELECT e FROM Employee e WHERE e.salary > ANY " +
"(SELECT AVG(e2.salary) FROM Employee e2 GROUP BY e2.departmentId)"

// SOME — ANY와 동일
"SELECT e FROM Employee e WHERE e.salary > SOME " +
"(SELECT AVG(e2.salary) FROM Employee e2 GROUP BY e2.departmentId)"
```

### JPA (Criteria API)

```java
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<Employee> cq = cb.createQuery(Employee.class);
Root<Employee> e = cq.from(Employee.class);

Subquery<Double> sub = cq.subquery(Double.class);
Root<Employee> e2 = sub.from(Employee.class);
sub.select(cb.avg(e2.get("salary")))
   .groupBy(e2.get("departmentId"));

// ALL
cq.where(cb.gt(e.get("salary"), cb.all(sub)));

// ANY
cq.where(cb.gt(e.get("salary"), cb.any(sub)));

// SOME
cq.where(cb.gt(e.get("salary"), cb.some(sub)));
```

### TypeORM QueryBuilder

TypeORM은 ALL, ANY, SOME에 대한 **전용 메서드가 없다**.
문자열로 직접 작성해야 한다.

```typescript
const subQuery = dataSource
    .createQueryBuilder()
    .subQuery()
    .select("AVG(e2.salary)")
    .from(Employee, "e2")
    .groupBy("e2.departmentId")
    .getQuery()

// ALL
dataSource
    .createQueryBuilder(Employee, "e")
    .where(`e.salary > ALL ${subQuery}`)
    .getMany()

// ANY
dataSource
    .createQueryBuilder(Employee, "e")
    .where(`e.salary > ANY ${subQuery}`)
    .getMany()
```

> **참고**: TypeORM의 `Any` FindOperator(`find({ where: { col: Any([...]) } })`)는
> PostgreSQL `= ANY(ARRAY[...])` 용도이며, 서브쿼리 ANY와는 **다른 기능**이다.

---

## 3. `[NOT] IN (subquery)`

서브쿼리 결과 집합에 값이 **포함되는지** 확인.

### JPA (JPQL)

```java
// IN
"SELECT e FROM Employee e WHERE e.departmentId IN " +
"(SELECT d.id FROM Department d WHERE d.active = true)"

// NOT IN
"SELECT e FROM Employee e WHERE e.departmentId NOT IN " +
"(SELECT d.id FROM Department d WHERE d.active = true)"
```

### JPA (Criteria API)

```java
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<Employee> cq = cb.createQuery(Employee.class);
Root<Employee> e = cq.from(Employee.class);

Subquery<Long> sub = cq.subquery(Long.class);
Root<Department> d = sub.from(Department.class);
sub.select(d.get("id"))
   .where(cb.isTrue(d.get("active")));

// IN
cq.where(e.get("departmentId").in(sub));

// NOT IN
cq.where(cb.not(e.get("departmentId").in(sub)));
```

### TypeORM QueryBuilder

```typescript
const subQuery = dataSource
    .createQueryBuilder()
    .subQuery()
    .select("d.id")
    .from(Department, "d")
    .where("d.active = true")
    .getQuery()

// IN
dataSource
    .createQueryBuilder(Employee, "e")
    .where(`e.departmentId IN ${subQuery}`)
    .getMany()

// NOT IN
dataSource
    .createQueryBuilder(Employee, "e")
    .where(`e.departmentId NOT IN ${subQuery}`)
    .getMany()
```

---

## 서브쿼리 파라미터 바인딩

### JPA

파라미터 바인딩이 자동으로 관리된다.

```java
// JPQL — 이름 기반
em.createQuery(
    "SELECT e FROM Employee e WHERE e.departmentId IN " +
    "(SELECT d.id FROM Department d WHERE d.size > :minSize)")
  .setParameter("minSize", 10)
  .getResultList();

// Criteria API — 자동 바인딩
Subquery<Long> sub = cq.subquery(Long.class);
Root<Department> d = sub.from(Department.class);
sub.select(d.get("id"))
   .where(cb.gt(d.get("size"), cb.parameter(Integer.class, "minSize")));
```

### TypeORM

서브쿼리와 메인 쿼리의 파라미터를 **수동으로 합쳐야** 한다.

```typescript
// 방법 1: subQuery() + getQuery() — 파라미터가 분리됨
const sub = dataSource
    .createQueryBuilder()
    .subQuery()
    .select("d.id")
    .from(Department, "d")
    .where("d.size > :minSize", { minSize: 10 })
    .getQuery()

dataSource
    .createQueryBuilder(Employee, "e")
    .where(`e.departmentId IN ${sub}`)
    .setParameters({ minSize: 10 }) // 서브쿼리의 파라미터를 다시 설정해야 함
    .getMany()

// 방법 2: whereExists() — 파라미터 자동 전파
// getExistsCondition()이 query.getParameters()를 반환하여 자동 합침
```

`whereExists()`를 사용하면 파라미터가 자동으로 전파되지만,
문자열 조합 방식(`subQuery().getQuery()`)에서는 수동으로 파라미터를 설정해야 한다.

---

## SELECT 절 서브쿼리

### Hibernate (JPA 표준 아님)

```java
// Hibernate 확장 — SELECT 절 스칼라 서브쿼리
"SELECT e, (SELECT COUNT(p) FROM Project p WHERE p.leaderId = e.id) " +
"FROM Employee e"
```

JPQL 표준에서는 불가하지만, Hibernate에서는 SELECT 절의 스칼라 서브쿼리를 지원한다.

### TypeORM QueryBuilder

```typescript
dataSource
    .createQueryBuilder(Employee, "e")
    .addSelect((subQuery) => {
        return subQuery
            .select("COUNT(p.id)")
            .from(Project, "p")
            .where("p.leaderId = e.id")
    }, "projectCount")
    .getMany()
```

TypeORM은 `.addSelect()` 콜백으로 SELECT 절 서브쿼리를 지원한다.
콜백 방식은 파라미터가 자동으로 전파되는 장점이 있다.

---

## FROM 절 서브쿼리 (인라인 뷰)

### JPA / Hibernate

```
JPQL/Hibernate 모두 FROM 절 서브쿼리를 지원하지 않는다.
→ JOIN으로 풀 수 있으면 조인으로 해결하는 것이 원칙.
```

### TypeORM QueryBuilder

```typescript
dataSource
    .createQueryBuilder()
    .select("sub.departmentId", "departmentId")
    .addSelect("sub.avgSalary", "avgSalary")
    .from((subQuery) => {
        return subQuery
            .select("e.departmentId", "departmentId")
            .addSelect("AVG(e.salary)", "avgSalary")
            .from(Employee, "e")
            .groupBy("e.departmentId")
    }, "sub")
    .getRawMany()
```

TypeORM은 `.from()` 콜백으로 FROM 절 서브쿼리를 지원한다.
JPQL에서 불가능한 인라인 뷰를 TypeORM에서는 사용할 수 있다.

---

## 핵심 비교 요약

| 기능                    | JPA (JPQL/Criteria)                 | TypeORM QueryBuilder              |
| ----------------------- | ----------------------------------- | --------------------------------- |
| `EXISTS`                | `cb.exists(sub)`                    | `.whereExists(sub)` (**메서드**)  |
| `NOT EXISTS`            | `cb.not(cb.exists(sub))`            | 문자열 조합 (메서드 없음)         |
| `ALL (subquery)`        | `cb.all(sub)`                       | 문자열 조합 (메서드 없음)         |
| `ANY / SOME (subquery)` | `cb.any(sub)` / `cb.some(sub)`      | 문자열 조합 (메서드 없음)         |
| `IN (subquery)`         | `path.in(sub)`                      | 문자열 조합                       |
| `NOT IN (subquery)`     | `cb.not(path.in(sub))`              | 문자열 조합                       |
| 파라미터 바인딩         | 자동                                | `whereExists`만 자동, 나머지 수동 |
| WHERE/HAVING 서브쿼리   | O (JPQL 표준)                       | O                                 |
| SELECT 서브쿼리         | X (Hibernate만 O)                   | O (`.addSelect()` 콜백)           |
| FROM 서브쿼리           | X                                   | O (`.from()` 콜백)                |
| 서브쿼리 허용 위치      | WHERE, HAVING (표준) + SELECT (HQL) | WHERE, HAVING, SELECT, FROM, JOIN |

### 핵심 차이

1. **JPA는 서브쿼리 위치가 제한적**이다 (WHERE, HAVING만). TypeORM은 SELECT, FROM, JOIN까지 가능.
2. **JPA Criteria API는 타입 안전한 서브쿼리 빌더**를 제공한다. TypeORM은 대부분 문자열 조합.
3. **TypeORM은 `whereExists()`만 전용 메서드**로 제공하고, 나머지는 모두 문자열로 작성해야 한다.
4. **파라미터 관리**: JPA는 자동, TypeORM은 콜백 방식(`.addSelect()`, `.from()`)에서만 자동.
5. **FROM 절 서브쿼리가 필요하면** JPQL에서는 JOIN으로 풀어야 하고, TypeORM에서는 직접 사용 가능.

---

## 설계 개선 이슈

위 분석에서 도출된 TypeORM 개선사항을 upstream에 이슈로 등록했다.

| 이슈                                                      | 개선 내용                                                                     | 현재 상태                                              |
| --------------------------------------------------------- | ----------------------------------------------------------------------------- | ------------------------------------------------------ |
| [#12065](https://github.com/typeorm/typeorm/issues/12065) | `whereNotExists()` / `andWhereNotExists()` / `orWhereNotExists()` 메서드 추가 | `whereExists()`는 있지만 NOT 대응 메서드 없음          |
| [#12066](https://github.com/typeorm/typeorm/issues/12066) | `whereAll()` / `whereAny()` / `whereSome()` 서브쿼리 메서드 추가              | 전용 메서드 없이 문자열 조합만 가능                    |
| [#12067](https://github.com/typeorm/typeorm/issues/12067) | `whereInSubquery()` / `whereNotInSubquery()` 메서드 + 파라미터 자동 전파      | `IN (subQuery)` 문자열 조합 시 파라미터 수동 관리 필요 |

공통 개선 방향: `whereExists()`가 `getExistsCondition()`으로 파라미터를 자동 전파하는 패턴을
나머지 서브쿼리 연산(`NOT EXISTS`, `ALL/ANY/SOME`, `IN/NOT IN`)에도 확장한다.
