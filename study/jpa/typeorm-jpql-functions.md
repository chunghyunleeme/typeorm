# JPQL 기본 함수

## 개요

JPQL은 데이터베이스에 독립적인 **표준 함수**를 제공한다.
TypeORM은 이들에 대한 **전용 메서드가 없으며**, raw SQL 문자열이나 `Raw` FindOperator로 작성해야 한다.

---

## 1. 문자열 함수

### CONCAT — 문자열 연결

```java
// JPQL
"SELECT CONCAT(e.firstName, ' ', e.lastName) FROM Employee e"
```

```java
// Criteria API
cb.concat(cb.concat(e.get("firstName"), " "), e.get("lastName"));
```

```typescript
// TypeORM QueryBuilder — raw SQL
qb.select("CONCAT(e.firstName, ' ', e.lastName)", "fullName").getRawMany()

// TypeORM Find API — Raw 연산자
repo.find({
    where: {
        name: Raw((col) => `CONCAT(${col}, '_suffix') = :val`, {
            val: "Tom_suffix",
        }),
    },
})
```

> **내부 사용**: TypeORM은 MSSQL/Spanner에서 복합 PK의 DISTINCT COUNT 시
> `CONCAT(col1, col2)` 형태로 내부적으로 사용한다.
> (`SelectQueryBuilder.ts:3158`)

---

### SUBSTRING — 부분 문자열 추출

```java
// JPQL — 1-based index
"SELECT SUBSTRING(e.name, 1, 3) FROM Employee e"
```

```java
// Criteria API
cb.substring(e.get("name"), 1, 3);
```

```typescript
// TypeORM — raw SQL
qb.select("SUBSTRING(e.name, 1, 3)", "prefix").getRawMany()
```

---

### TRIM — 공백 제거

```java
// JPQL
"SELECT TRIM(e.name) FROM Employee e"
"SELECT TRIM(LEADING ' ' FROM e.name) FROM Employee e"   // 앞쪽만
"SELECT TRIM(TRAILING ' ' FROM e.name) FROM Employee e"  // 뒤쪽만
"SELECT TRIM(BOTH ' ' FROM e.name) FROM Employee e"      // 양쪽
```

```java
// Criteria API
cb.trim(e.get("name"));
cb.trim(Trimspec.LEADING, e.get("name"));
```

```typescript
// TypeORM — raw SQL
qb.select("TRIM(e.name)", "trimmedName")
qb.select("TRIM(LEADING ' ' FROM e.name)", "ltrimmed")
```

---

### LOWER, UPPER — 대소문자 변환

```java
// JPQL
"SELECT LOWER(e.name) FROM Employee e"
"SELECT UPPER(e.name) FROM Employee e"
```

```java
// Criteria API
cb.lower(e.get("name"));
cb.upper(e.get("name"));
```

```typescript
// TypeORM — raw SQL
qb.select("LOWER(e.name)", "lowerName")
qb.select("UPPER(e.name)", "upperName")

// Find API — ILike가 내부적으로 UPPER 사용
repo.find({ where: { name: ILike("%tom%") } })
// → UPPER(name) LIKE UPPER('%tom%')
```

> **내부 사용**: `ILike` FindOperator가 내부적으로 `UPPER(column) LIKE UPPER(value)`로
> 변환한다. (`QueryBuilder.ts:1135`)

---

### LENGTH — 문자열 길이

```java
// JPQL
"SELECT LENGTH(e.name) FROM Employee e"
```

```java
// Criteria API
cb.length(e.get("name"));
```

```typescript
// TypeORM — raw SQL
qb.select("LENGTH(e.name)", "nameLength")
qb.where("LENGTH(e.name) > :min", { min: 5 })
```

---

### LOCATE — 문자열 위치 검색

```java
// JPQL — 1-based, 못 찾으면 0
"SELECT LOCATE('o', e.name) FROM Employee e"
"SELECT LOCATE('o', e.name, 3) FROM Employee e"  // 3번째 위치부터 검색
```

```java
// Criteria API
cb.locate(e.get("name"), "o");
cb.locate(e.get("name"), "o", 3);
```

```typescript
// TypeORM — raw SQL
qb.select("LOCATE('o', e.name)", "pos")
// MySQL: LOCATE('o', name)
// PostgreSQL: POSITION('o' IN name) 또는 STRPOS(name, 'o')
```

> **주의**: `LOCATE`는 데이터베이스마다 문법이 다를 수 있다.
> JPA는 JPQL이 이를 추상화하지만, TypeORM은 raw SQL이므로 DB별 차이를 직접 처리해야 한다.

---

## 2. 수학 함수

### ABS — 절대값

```java
// JPQL
"SELECT ABS(e.balance) FROM Employee e"
```

```java
// Criteria API
cb.abs(e.get("balance"));
```

```typescript
// TypeORM — raw SQL
qb.select("ABS(e.balance)", "absBalance")
```

---

### SQRT — 제곱근

```java
// JPQL
"SELECT SQRT(e.area) FROM Employee e"
```

```java
// Criteria API
cb.sqrt(e.get("area"));
```

```typescript
// TypeORM — raw SQL
qb.select("SQRT(e.area)", "sqrtArea")
```

---

### MOD — 나머지

```java
// JPQL
"SELECT MOD(e.id, 2) FROM Employee e"  // 홀짝 판별
```

```java
// Criteria API
cb.mod(e.get("id"), 2);
```

```typescript
// TypeORM — raw SQL
qb.select("MOD(e.id, 2)", "remainder")
// 또는
qb.where("MOD(e.id, 2) = 0") // 짝수만
```

---

## 3. 컬렉션 함수 (JPA 전용)

이 두 함수는 **JPA의 엔티티 컬렉션 개념**에 의존하며, TypeORM에는 대응하는 기능이 없다.

### SIZE — 컬렉션 크기

연관된 컬렉션의 **요소 개수**를 반환한다.

```java
// JPQL — Team의 members 컬렉션 크기
"SELECT t FROM Team t WHERE SIZE(t.members) > 5"
```

```java
// Criteria API
cb.size(t.get("members"));
```

JPA는 `SIZE(t.members)`를 내부적으로 **COUNT 서브쿼리**로 변환한다:

```sql
-- 실제 생성 SQL
SELECT t.* FROM team t
WHERE (SELECT COUNT(*) FROM member m WHERE m.team_id = t.id) > 5
```

```typescript
// TypeORM — 직접 서브쿼리로 작성해야 함
dataSource
    .createQueryBuilder(Team, "t")
    .where((qb) => {
        const sub = qb
            .subQuery()
            .select("COUNT(m.id)")
            .from(Member, "m")
            .where("m.teamId = t.id")
            .getQuery()
        return `${sub} > 5`
    })
    .getMany()

// 또는 loadRelationCountAndMap 사용
dataSource
    .createQueryBuilder(Team, "t")
    .loadRelationCountAndMap("t.memberCount", "t.members")
    .getMany()
```

> **TypeORM 대안**: `loadRelationCountAndMap()`으로 관계 카운트를 매핑할 수 있지만,
> WHERE 절 조건으로 사용할 수는 없다.

---

### INDEX — 순서 컬렉션의 인덱스

`@OrderColumn`으로 정렬된 리스트에서 **요소의 위치**를 반환한다.

```java
// JPA 엔티티 — 순서가 있는 컬렉션
@Entity
public class Team {
    @OneToMany
    @OrderColumn(name = "position")  // 순서 컬럼
    private List<Member> members;
}

// JPQL — 인덱스 기반 조회
"SELECT m FROM Team t JOIN t.members m WHERE INDEX(m) = 0"  // 첫 번째 멤버
```

```typescript
// TypeORM — @OrderColumn 개념 없음
// 순서가 필요하면 엔티티에 position 컬럼을 직접 정의해야 함
@Entity()
class Member {
    @Column()
    position: number // 수동 관리
}

dataSource.createQueryBuilder(Member, "m").where("m.position = 0").getMany()
```

> **핵심 차이**: JPA의 `INDEX()`는 `@OrderColumn`과 연동되어 컬렉션의 순서를 투명하게 관리한다.
> TypeORM에는 `@OrderColumn`이 없으므로 순서 컬럼을 수동으로 정의하고 관리해야 한다.

---

## 핵심 비교 요약

### 문자열 / 수학 함수

| 함수        | JPA (JPQL)           | JPA (Criteria API)      | TypeORM              |
| ----------- | -------------------- | ----------------------- | -------------------- |
| `CONCAT`    | `CONCAT(a, b)`       | `cb.concat(a, b)`       | raw SQL              |
| `SUBSTRING` | `SUBSTRING(s, i, l)` | `cb.substring(s, i, l)` | raw SQL              |
| `TRIM`      | `TRIM(s)`            | `cb.trim(s)`            | raw SQL              |
| `LOWER`     | `LOWER(s)`           | `cb.lower(s)`           | raw SQL (ILike 내부) |
| `UPPER`     | `UPPER(s)`           | `cb.upper(s)`           | raw SQL (ILike 내부) |
| `LENGTH`    | `LENGTH(s)`          | `cb.length(s)`          | raw SQL              |
| `LOCATE`    | `LOCATE(s, sub)`     | `cb.locate(s, sub)`     | raw SQL (DB별 차이)  |
| `ABS`       | `ABS(n)`             | `cb.abs(n)`             | raw SQL              |
| `SQRT`      | `SQRT(n)`            | `cb.sqrt(n)`            | raw SQL              |
| `MOD`       | `MOD(a, b)`          | `cb.mod(a, b)`          | raw SQL              |

### 컬렉션 함수 (JPA 전용)

| 함수    | JPA                            | TypeORM 대안                                  |
| ------- | ------------------------------ | --------------------------------------------- |
| `SIZE`  | `SIZE(t.members)` → COUNT 변환 | 서브쿼리 또는 `loadRelationCountAndMap()`     |
| `INDEX` | `INDEX(m)` + `@OrderColumn`    | position 컬럼 수동 정의 (`@OrderColumn` 없음) |

### 핵심 차이

1. **JPQL 표준 함수는 DB 독립적**이다. JPA가 DB별 SQL로 변환해준다.
   TypeORM은 raw SQL이므로 `LOCATE` 같은 함수는 **DB별 문법 차이를 직접 처리**해야 한다.
2. **Criteria API는 모든 함수에 타입 안전한 메서드**를 제공한다.
   TypeORM은 전용 메서드가 없고 문자열로만 사용 가능하다.
3. **SIZE, INDEX는 JPA의 엔티티 컬렉션 모델**에 의존하는 함수다.
   TypeORM에는 이 개념이 없어 서브쿼리나 수동 컬럼으로 대체해야 한다.
4. TypeORM에서 유일하게 **함수를 내부적으로 사용하는 경우**:
    - `ILike` → `UPPER()` 변환
    - MSSQL/Spanner DISTINCT COUNT → `CONCAT()` 사용

---

## 설계 개선 이슈

위 분석에서 도출된 TypeORM 개선사항을 upstream에 이슈로 등록했다.

| 이슈                                                      | 개선 내용                                                | 현재 상태                                               |
| --------------------------------------------------------- | -------------------------------------------------------- | ------------------------------------------------------- |
| [#12072](https://github.com/typeorm/typeorm/issues/12072) | DB 독립적 SQL 함수 헬퍼 추가 (`locate()`, `concat()` 등) | LOCATE 등 DB별 문법 차이를 사용자가 직접 처리해야 함    |
| [#12073](https://github.com/typeorm/typeorm/issues/12073) | 관계 카운트 WHERE 조건 지원 (JPA `SIZE()` 대응)          | `loadRelationCountAndMap()`은 SELECT만 가능, WHERE 불가 |
| [#12074](https://github.com/typeorm/typeorm/issues/12074) | `@OrderColumn` 데코레이터 추가 (JPA `INDEX()` 대응)      | 순서 컬럼 수동 정의·관리 필요, 자동 순서 관리 없음      |

공통 개선 방향: JPA가 JPQL 표준 함수로 **DB 독립적 추상화**와 **엔티티 컬렉션 모델 연동**을
제공하는 것처럼, TypeORM도 자주 사용되는 함수와 컬렉션 패턴에 대한 전용 API를 제공한다.
