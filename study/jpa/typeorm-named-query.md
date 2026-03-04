# Named Query

## 개요

JPA의 `@NamedQuery`는 **엔티티 클래스에 미리 정의해두는 정적 JPQL 쿼리**다.
애플리케이션 로딩 시점에 파싱·검증되므로, 런타임 오류를 사전에 방지할 수 있다.

TypeORM에는 `@NamedQuery` 개념이 없다. 대신 **Custom Repository 패턴**으로 재사용 가능한 쿼리를 조직한다.

---

## JPA @NamedQuery

### 정의

```java
@Entity
@NamedQuery(
    name = "Employee.findByDepartment",
    query = "SELECT e FROM Employee e WHERE e.department = :dept"
)
@NamedQuery(
    name = "Employee.findHighSalary",
    query = "SELECT e FROM Employee e WHERE e.salary > :threshold ORDER BY e.salary DESC"
)
public class Employee {
    @Id @GeneratedValue
    private Long id;
    private String name;
    private String department;
    private int salary;
}
```

### 호출

```java
// 이름으로 호출
List<Employee> employees = em.createNamedQuery("Employee.findByDepartment", Employee.class)
    .setParameter("dept", "IT")
    .getResultList();

List<Employee> highPaid = em.createNamedQuery("Employee.findHighSalary", Employee.class)
    .setParameter("threshold", 10000)
    .getResultList();
```

### 핵심 특징

1. **로딩 시점 검증** — 애플리케이션 구동 시 JPQL 구문 오류를 즉시 발견
2. **사전 파싱** — JPA 구현체(Hibernate)가 미리 파싱하여 성능 최적화
3. **엔티티 중심 조직** — 쿼리가 엔티티 클래스에 선언되어 관련 쿼리를 한눈에 파악
4. **전역 네임스페이스** — 이름이 persistence unit 내에서 유일해야 함

---

## JPA @NamedNativeQuery

네이티브 SQL도 Named Query로 정의할 수 있다.

```java
@Entity
@NamedNativeQuery(
    name = "Employee.findByDeptNative",
    query = "SELECT * FROM employees WHERE department = ?",
    resultClass = Employee.class
)
public class Employee { ... }
```

```java
List<Employee> result = em.createNamedQuery("Employee.findByDeptNative", Employee.class)
    .setParameter(1, "IT")
    .getResultList();
```

---

## TypeORM — @NamedQuery 없음

TypeORM에는 엔티티에 쿼리를 선언하는 메커니즘이 없다.
대신 아래 패턴들로 재사용 가능한 쿼리를 조직한다.

### 1. Custom Repository (권장 패턴)

`Repository.extend()`로 커스텀 메서드를 추가한다.

```typescript
// employee.repository.ts
export const EmployeeRepository = dataSource.getRepository(Employee).extend({
    findByDepartment(dept: string) {
        return this.createQueryBuilder("e")
            .where("e.department = :dept", { dept })
            .getMany()
    },

    findHighSalary(threshold: number) {
        return this.createQueryBuilder("e")
            .where("e.salary > :threshold", { threshold })
            .orderBy("e.salary", "DESC")
            .getMany()
    },
})

// 호출
const itEmployees = await EmployeeRepository.findByDepartment("IT")
const highPaid = await EmployeeRepository.findHighSalary(10000)
```

### 2. Find API 활용

간단한 조건은 Find API만으로 충분하다.

```typescript
export const EmployeeRepository = dataSource.getRepository(Employee).extend({
    findByDepartment(dept: string) {
        return this.find({
            where: { department: dept },
        })
    },

    findHighSalary(threshold: number) {
        return this.find({
            where: { salary: MoreThan(threshold) },
            order: { salary: "DESC" },
        })
    },
})
```

### 3. 쿼리 캐싱 (Named Cache)

TypeORM은 쿼리 결과 캐싱을 지원한다.
이름을 붙여 캐시를 관리할 수 있다.

```typescript
// 이름 기반 캐시 — 25초 동안 결과 재사용
const admins = await dataSource
    .createQueryBuilder(Employee, "e")
    .where("e.isAdmin = true")
    .cache("employees_admins", 25000)
    .getMany()

// 특정 캐시 무효화
await dataSource.queryResultCache.remove(["employees_admins"])

// Find API에서도 캐시 사용 가능
const result = await dataSource.getRepository(Employee).find({
    where: { isAdmin: true },
    cache: { id: "employees_admins", milliseconds: 25000 },
})
```

---

## 트랜잭션에서의 Custom Repository

Custom Repository는 트랜잭션 내에서도 사용할 수 있다.

### JPA

```java
// @NamedQuery는 트랜잭션과 무관하게 동일하게 동작
@Transactional
public void process() {
    List<Employee> employees = em.createNamedQuery("Employee.findByDepartment", Employee.class)
        .setParameter("dept", "IT")
        .getResultList();
    // ...
}
```

### TypeORM

```typescript
// withRepository()로 트랜잭션 매니저에 바인딩
await dataSource.transaction(async (manager) => {
    const txRepo = manager.withRepository(EmployeeRepository)
    const employees = await txRepo.findByDepartment("IT")
    // txRepo의 모든 쿼리가 이 트랜잭션 내에서 실행됨
})
```

---

## Spring Data JPA와의 비교

실무에서 `@NamedQuery`보다 더 많이 사용하는 것은 **Spring Data JPA의 메서드 이름 기반 쿼리**다.

### Spring Data JPA

```java
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
    // 메서드 이름에서 쿼리 자동 생성
    List<Employee> findByDepartment(String dept);
    List<Employee> findBySalaryGreaterThanOrderBySalaryDesc(int threshold);

    // 복잡한 쿼리는 @Query 사용
    @Query("SELECT e FROM Employee e WHERE e.department = :dept AND e.salary > :min")
    List<Employee> findByDeptAndMinSalary(@Param("dept") String dept, @Param("min") int min);
}
```

### TypeORM Custom Repository

```typescript
export const EmployeeRepository = dataSource.getRepository(Employee).extend({
    // 직접 메서드 구현 — 이름 컨벤션 자동 생성 없음
    findByDepartment(dept: string) {
        return this.find({ where: { department: dept } })
    },

    findByDeptAndMinSalary(dept: string, min: number) {
        return this.createQueryBuilder("e")
            .where("e.department = :dept", { dept })
            .andWhere("e.salary > :min", { min })
            .getMany()
    },
})
```

---

## 핵심 비교 요약

| 항목               | JPA @NamedQuery                      | TypeORM Custom Repository          |
| ------------------ | ------------------------------------ | ---------------------------------- |
| **쿼리 정의 위치** | 엔티티 클래스 (`@NamedQuery`)        | 별도 파일 (`Repository.extend()`)  |
| **구문 검증 시점** | 애플리케이션 로딩 시 (사전 파싱)     | 런타임 실행 시                     |
| **쿼리 식별**      | 문자열 이름 (`"Employee.findBy..."`) | 메서드 이름 (`findByDepartment()`) |
| **네이티브 SQL**   | `@NamedNativeQuery`                  | `manager.query()` 또는 raw SQL     |
| **캐싱**           | JPA 구현체 레벨 (Hibernate 2차 캐시) | `.cache("name", ms)` 내장          |
| **트랜잭션 사용**  | 투명 (EntityManager가 관리)          | `manager.withRepository()` 필요    |
| **자동 쿼리 생성** | Spring Data JPA `findBy...`          | 없음 (직접 구현)                   |
| **타입 안전성**    | 제네릭 반환 타입                     | TypeScript 타입 추론               |

### 핵심 차이

1. **JPA @NamedQuery의 최대 장점은 로딩 시점 검증**이다.
   JPQL 구문 오류를 배포 전에 발견할 수 있다.
   TypeORM은 런타임에서야 쿼리 오류를 발견한다.

2. **TypeORM Custom Repository는 코드 기반**이므로 IDE 자동완성과 리팩토링이 용이하다.
   JPA @NamedQuery는 문자열 기반이라 이름 변경 시 호출부를 수동으로 찾아야 한다.

3. **실무에서 @NamedQuery는 잘 사용하지 않는다.**
   Spring Data JPA의 `@Query`나 메서드 이름 기반 쿼리가 더 편리하기 때문이다.
   TypeORM의 Custom Repository 패턴이 Spring Data JPA의 커스텀 메서드와 가장 유사하다.

4. **쿼리 캐싱 방식이 다르다.**
   JPA는 Hibernate 2차 캐시(엔티티 캐시, 쿼리 캐시)로 다층 캐싱을 제공하고,
   TypeORM은 `.cache()` 메서드로 쿼리 결과를 DB 테이블 또는 Redis에 캐싱한다.
