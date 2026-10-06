# DB 마이그레이션 도구: Flyway / Liquibase / Prisma

## 왜 마이그레이션 도구를 쓰나

스키마 변경(DDL)을 **코드처럼 버전 관리**하기 위해서.

```
개발자 A: ALTER TABLE 직접 실행 → 로컬 DB만 바뀜
개발자 B: 모름 → 로컬에서 컬럼 없다고 에러
운영 배포: 누가 어떤 DDL을 돌렸는지 아무도 모름  ← 사고
```

마이그레이션 도구는 공통적으로 이렇게 동작한다.

1. 변경 스크립트를 **파일로** 저장 (git에 커밋)
2. DB 안에 **이력 테이블**을 만들어 어디까지 적용했는지 기록
3. 실행 시 **아직 적용 안 된 것만** 순서대로 적용

| 도구 | 이력 테이블 |
|------|------------|
| Flyway | `flyway_schema_history` |
| Liquibase | `DATABASECHANGELOG`, `DATABASECHANGELOGLOCK` |
| Prisma | `_prisma_migrations` |

---

## Flyway

**SQL 파일 + 파일명 규칙**이 전부. 가장 단순하다.

### 파일명 규칙

```
src/main/resources/db/migration/
├── V1__create_member.sql        ← Versioned: 1번만 실행
├── V2__add_member_email.sql
├── V2_1__add_index.sql          ← 버전은 2.1
└── R__member_view.sql           ← Repeatable: 내용(checksum) 바뀔 때마다 재실행
```

- `V{버전}__{설명}.sql` — 언더스코어 **2개** 주의. 1개면 마이그레이션으로 인식 안 되고 **에러 없이 무시**된다 (`validateMigrationNaming=true`로 두면 에러)
- `R__` — 뷰, 프로시저, 함수처럼 "덮어쓰기" 가능한 객체용. Versioned 다 끝난 뒤 실행
- `U{버전}__` — Undo 마이그레이션. 같은 버전의 `V`를 되돌리는 SQL을 직접 작성

```sql
-- V2__add_member_email.sql
ALTER TABLE member ADD COLUMN email VARCHAR(255);
CREATE UNIQUE INDEX uk_member_email ON member (email);
```

### Spring Boot 연동

```kotlin
// build.gradle.kts — Spring Boot 4.x
implementation("org.springframework.boot:spring-boot-starter-flyway")
implementation("org.flywaydb:flyway-mysql")   // DB별 모듈 (PostgreSQL은 flyway-database-postgresql)

// Spring Boot 3.x
implementation("org.flywaydb:flyway-core")
implementation("org.flywaydb:flyway-mysql")
```

- Spring Boot 4부터 자동 설정이 모듈로 쪼개져서 `flyway-core`만 넣으면 **기동 시 migrate가 안 돈다.** starter 필요
- Flyway 10부터 MySQL, PostgreSQL, Oracle, SQL Server 등은 **DB별 모듈을 따로** 넣어야 한다

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration   # 기본값
    baseline-on-migrate: true           # 이미 테이블 있는 기존 DB에 처음 도입할 때
  jpa:
    hibernate:
      ddl-auto: validate                # 스키마는 Flyway가, JPA는 검증만
```

의존성만 넣으면 **애플리케이션 기동 시 자동으로 migrate** 된다.

### 주요 명령

| 명령 | 하는 일 |
|------|--------|
| `migrate` | 미적용 마이그레이션 실행 |
| `info` | 적용/대기 상태 출력 |
| `validate` | 적용된 파일의 checksum이 그대로인지 검사 |
| `baseline` | 기존 DB를 "여기까지는 적용된 걸로 치자" 표시 |
| `repair` | 실패 기록 삭제, checksum 재정렬 |
| `clean` | **스키마 전체 DROP**. 운영 절대 금지 (Spring Boot는 기본 비활성화) |

### 주의: 적용된 파일은 절대 수정하지 않는다

```
V2__add_member_email.sql 을 적용 후 수정
→ 다음 기동 시 validate 실패
→ "Migration checksum mismatch for migration version 2"
→ 애플리케이션 기동 안 됨
```

잘못 만들었으면 **V3로 고치는 마이그레이션을 새로 추가**한다. (로컬 전용이면 `repair`로 checksum 재정렬 가능)

---

## Liquibase

**changelog → changeSet** 구조. DB 독립적인 포맷(XML/YAML/JSON) 또는 SQL로 작성.

### 구조

```
src/main/resources/db/changelog/
├── db.changelog-master.yaml      ← 진입점 (Spring Boot 기본 경로)
└── changes/
    ├── 001-create-member.yaml
    └── 002-add-member-email.yaml
```

```yaml
# db.changelog-master.yaml
databaseChangeLog:
  - include:
      file: db/changelog/changes/001-create-member.yaml
  - include:
      file: db/changelog/changes/002-add-member-email.yaml
```

```yaml
# 002-add-member-email.yaml
databaseChangeLog:
  - changeSet:
      id: 002-add-member-email
      author: anjunggeon
      changes:
        - addColumn:
            tableName: member
            columns:
              - column:
                  name: email
                  type: varchar(255)
      rollback:
        - dropColumn:
            tableName: member
            columnName: email
```

changeSet은 **`id` + `author` + 파일 경로** 조합으로 식별한다. 파일명이 아니라 changeSet 단위로 이력이 쌓인다.

SQL이 편하면 Formatted SQL도 된다.

```sql
--liquibase formatted sql

--changeset anjunggeon:002-add-member-email
ALTER TABLE member ADD COLUMN email VARCHAR(255);
--rollback ALTER TABLE member DROP COLUMN email;
```

### Spring Boot 연동

```kotlin
// Spring Boot 4.x
implementation("org.springframework.boot:spring-boot-starter-liquibase")

// Spring Boot 3.x
implementation("org.liquibase:liquibase-core")
```

```yaml
spring:
  liquibase:
    change-log: classpath:db/changelog/db.changelog-master.yaml
    contexts: prod            # context가 prod인 changeSet + context 없는 changeSet 실행
```

context 주의점 (헷갈리기 쉬움):

| 실행 시 contexts | context 없는 changeSet | `context: dev` | `context: "@dev"` |
|-----------------|----------------------|----------------|-------------------|
| 지정 안 함 | 실행 | **실행** | 실행 안 함 |
| `prod` | 실행 | 실행 안 함 | 실행 안 함 |
| `dev` | 실행 | 실행 | 실행 |

- context를 안 붙인 changeSet은 **항상** 실행된다
- 실행할 때 contexts를 비워두면 `context: dev`도 **전부 실행**된다 → 운영에서 테스트 데이터가 들어가는 사고
- `@`를 붙이면 그 context를 명시했을 때만 실행 (지정 안 하면 건너뜀)

### 주요 기능

| 기능 | 설명 | Flyway에서는 |
|------|------|-------------|
| **rollback** | `addColumn`, `createTable` 등은 롤백 자동 생성. SQL은 `rollback` 블록 직접 작성 | `U__` Undo 파일 직접 작성 |
| **contexts / labels** | `context: dev` 처럼 환경별로 changeSet 실행 여부 제어 (테스트 데이터 등) | 없음 (`locations`를 환경별로 나눠 우회) |
| **preconditions** | "테이블이 없을 때만 실행" 같은 조건 | 없음 (SQL의 `IF NOT EXISTS`로 우회) |
| **diff / generate-changelog** | 두 DB 비교, 기존 DB에서 changelog 역생성 | `diff` / `generate` |
| **update-sql** | 실제 실행 없이 실행될 SQL만 출력 (운영 반영 전 DBA 검토용) | dry run |

### 재실행 옵션 (Flyway에도 대응 기능 있음)

| Liquibase | Flyway 대응 | 실행 조건 |
|-----------|------------|----------|
| `runOnChange: true` | `R__` 파일 | checksum이 바뀔 때만 재실행 |
| `runAlways: true` | `afterMigrate.sql` 콜백 | migrate할 때마다 무조건 실행 |

```yaml
- changeSet:
    id: member-view
    author: anjunggeon
    runOnChange: true
    changes:
      - sql:
          sql: CREATE OR REPLACE VIEW v_member AS SELECT id, name FROM member
```

차이: Flyway의 `R__`, `afterMigrate.sql`은 Versioned가 **전부 끝난 뒤** 실행되고, Liquibase의 `runOnChange`, `runAlways`는 **changelog 안의 자기 위치**에서 실행된다.
공통: 재실행되므로 `CREATE OR REPLACE`처럼 **여러 번 돌려도 안전한 SQL**만 넣는다. (`createTable`에 붙이면 두 번째 실행에서 터짐)

### 주의

- 적용된 changeSet 수정 시 checksum 불일치로 실패하는 건 Flyway와 동일
- 기동 중 비정상 종료되면 `DATABASECHANGELOGLOCK`에 락이 남아 다음 기동이 락을 기다리다 실패함 → `release-locks` 명령 또는 `LOCKED = 0`으로 수동 해제

---

## Prisma (Prisma Migrate)

앞의 둘과 성격이 다르다. **Node.js/TypeScript ORM**이고, 마이그레이션은 그 일부 기능.
SQL을 직접 쓰는 게 아니라 **스키마를 선언하면 SQL을 생성**해준다. (JPA `ddl-auto`를 파일로 남기는 느낌)

### 스키마 선언

```ts
// prisma.config.ts (Prisma 7~) — DB 접속 정보는 여기
import "dotenv/config";   // v7부터 .env 자동 로딩 안 함
import { defineConfig, env } from "prisma/config";

export default defineConfig({
  schema: "prisma/schema.prisma",
  migrations: { path: "prisma/migrations" },
  datasource: { url: env("DATABASE_URL") },
});
```

```prisma
// prisma/schema.prisma
datasource db {
  provider = "postgresql"          // v6까지는 여기에 url = env("DATABASE_URL")
}

model Member {
  id    Int     @id @default(autoincrement())
  name  String
  email String? @unique              // ← 이 줄 추가 후 migrate dev
}
```

### 흐름

```bash
npx prisma migrate dev --name add_member_email
```

```
prisma/migrations/
├── 20261006120000_init/migration.sql
├── 20261006130000_add_member_email/migration.sql   ← 자동 생성된 SQL
└── migration_lock.toml
```

생성된 `migration.sql`은 git에 커밋하고, 필요하면 **적용 전에 직접 수정**할 수 있다 (`--create-only`로 생성만 하고 멈춤).

### 주요 명령

| 명령 | 환경 | 하는 일 |
|------|------|--------|
| `migrate dev` | 개발 | 스키마 diff → SQL 생성 → 적용. shadow DB로 drift 검사 |
| `migrate deploy` | **운영/CI** | 미적용 마이그레이션만 적용. 생성/리셋 안 함 |
| `migrate reset` | 개발 | DB 날리고 마이그레이션 전부 재적용 |
| `generate` | - | Prisma Client(쿼리용 코드) 생성 |
| `db seed` | 개발 | 초기 데이터 입력 |
| `migrate resolve` | 운영 | 실패한 마이그레이션을 applied / rolled-back으로 표시, baseline 시에도 사용 |
| `migrate diff` | - | 두 상태 사이 SQL 생성 (down 스크립트 만들 때 활용) |
| `db push` | 프로토타입 | 마이그레이션 파일 없이 스키마를 DB에 바로 반영 |
| `db pull` | - | 기존 DB에서 schema.prisma 역생성 |

### 주의

- **Prisma 7부터 `migrate dev`가 `generate`, `seed`를 자동으로 안 돌린다.** 스키마를 바꿨으면 `prisma generate`를 따로 실행해야 Client 타입이 갱신된다 (v6까지는 자동)
- **운영에서 `migrate dev` 쓰면 안 된다** — 데이터 리셋을 제안할 수 있음. 운영은 무조건 `migrate deploy`
- 자동 down 마이그레이션 없음 → 되돌리려면 새 마이그레이션을 추가하거나 `migrate diff`로 down SQL 생성
- 컬럼 rename을 "삭제 + 추가"로 인식할 수 있음 → `--create-only`로 SQL 확인 후 `RENAME COLUMN`으로 직접 수정

---

## 비교

| 항목 | Flyway | Liquibase | Prisma Migrate |
|------|--------|-----------|----------------|
| 생태계 | JVM (Spring 기본 지원) | JVM (Spring 기본 지원) | Node.js / TypeScript |
| 작성 방식 | SQL 직접 | XML/YAML/JSON/SQL | 스키마 선언 → SQL 자동 생성 |
| 실행 단위 | 파일 | changeSet | 디렉터리(파일) |
| 순서 결정 | 파일명 버전 | changelog 기재 순서 | 디렉터리명 타임스탬프 |
| 롤백 | `U__` Undo 파일 직접 작성 | `rollback` 블록 (일부는 자동 생성) | 없음 (forward only) |
| DB 독립성 | 낮음 (SQL 방언 그대로) | 높음 (추상 포맷 사용 시) | 스키마는 독립적, 생성된 SQL은 DB 종속 (`migration_lock.toml`이 provider 고정) |
| 환경별 분기 | location 분리로 우회 | contexts / labels | 없음 |
| 학습 비용 | 가장 낮음 | 높음 | 낮음 (Prisma 쓰는 전제) |

---

## 언제 무엇을 쓰나

```
백엔드가 Node.js/TS이고 Prisma ORM을 쓰는가?
├── 예 → Prisma Migrate
└── 아니오 (Spring/JPA 등)
    │
    여러 종류의 DB를 지원해야 하거나,
    환경별 분기(contexts)·조건부 실행(preconditions)이 필요한가?
    ├── 예 → Liquibase
    └── 아니오 → Flyway (대부분의 경우 이걸로 충분)
```

---

## 공통 원칙

1. **적용된 마이그레이션은 수정하지 않는다.** 고칠 게 있으면 새 파일로
2. **운영에서 `ddl-auto: update`/`create` 쓰지 않는다.** 마이그레이션 도구 + `validate`
3. **파괴적 변경은 단계를 나눈다** (컬럼 rename 예시)
   ```
   1차 배포: 새 컬럼 추가 + 양쪽에 쓰기
   2차 배포: 데이터 이관, 읽기를 새 컬럼으로
   3차 배포: 옛 컬럼 삭제
   ```
   한 번에 rename하면 배포 중 구버전 인스턴스가 옛 컬럼을 찾다가 터진다
4. **큰 테이블 DDL은 락을 확인한다.** 마이그레이션이 기동 시 실행되면 락 걸린 동안 애플리케이션이 안 뜬다
5. 기존 DB에 처음 도입할 땐 **baseline**부터 (Flyway `baseline`, Liquibase `changelog-sync`, Prisma `migrate resolve --applied`)
