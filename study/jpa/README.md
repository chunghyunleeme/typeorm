# JPA vs TypeORM 비교 학습

JPA 개발자 관점에서 TypeORM 내부 동작을 분석하고 비교한 학습 노트 모음.

## 목차

| 파일 | 주제 |
|---|---|
| [typeorm-lazy-relation-internals.md](./typeorm-lazy-relation-internals.md) | Lazy Relation 내부 동작 원리 (데코레이터 → 메타데이터 빌드 → Object.defineProperty → 클로저) |
| [typeorm-architecture-layer.md](./typeorm-architecture-layer.md) | 아키텍처 계층 구조 비교 (JPA/JDBC 표준 스펙 vs TypeORM/npm 패키지) |