# Offset/Limit 페이지네이션과 커버링 인덱스

## Offset/Limit 기본

```sql
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 100;
```

- `LIMIT` : 가져올 행 수
- `OFFSET` : 건너뛸 행 수

### 문제: OFFSET이 커질수록 느려진다

DB는 OFFSET만큼 행을 **실제로 읽고 버린다.**  
`OFFSET 10000 LIMIT 20` → 내부적으로 10,020행 읽음.

```
page 1  → OFFSET 0    → 20행 읽음
page 100 → OFFSET 1980 → 2,000행 읽음
page 500 → OFFSET 9980 → 10,000행 읽음  ← 점점 느려짐
```

---

## 커버링 인덱스 (Covering Index)

쿼리가 필요한 컬럼이 **인덱스 안에 다 들어있어서** 테이블 본문(heap)을 전혀 안 읽는 경우.

```sql
-- 인덱스: (created_at, id)
SELECT id, created_at FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 100;
--  ↑ 두 컬럼 모두 인덱스에 있음 → heap 접근 없이 인덱스만으로 결과 반환
```

### MySQL EXPLAIN에서 확인
```
Extra: Using index   ← 커버링 인덱스 사용
Extra: (없음)        ← heap 읽음
```

---

## 커버링 인덱스로 OFFSET 느림 우회하기

`SELECT *` 는 커버링 인덱스를 쓸 수 없다. 대신 **id만 먼저 커버링 인덱스로 뽑고**, 그 id로 본문을 조인한다.

```sql
-- 느린 쿼리
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 10000;

-- 빠른 쿼리 (Deferred Join / Late Row Lookup)
SELECT o.*
FROM orders o
JOIN (
    SELECT id FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 10000
) sub ON o.id = sub.id;
```

서브쿼리는 `(created_at, id)` 인덱스만으로 실행(heap 없음) → 20개 id 확보 → 본문은 20행만 읽음.

### JPA/QueryDSL 적용 예시
```kotlin
// 1단계: 커버링 인덱스로 id만 추출
val ids = queryFactory
    .select(order.id)
    .from(order)
    .orderBy(order.createdAt.desc())
    .offset(pageable.offset)
    .limit(pageable.pageSize.toLong())
    .fetch()

// 2단계: id로 본문 조회
val result = queryFactory
    .selectFrom(order)
    .where(order.id.`in`(ids))
    .orderBy(order.createdAt.desc())
    .fetch()
```

---

## Keyset 페이지네이션 (Cursor 방식)

OFFSET을 아예 없애는 방법. "마지막으로 본 값보다 작은 것을 가져와."

```sql
-- 첫 페이지
SELECT * FROM orders ORDER BY created_at DESC, id DESC LIMIT 20;

-- 다음 페이지 (마지막 행의 created_at, id를 cursor로 사용)
SELECT * FROM orders
WHERE (created_at, id) < ('2024-01-15 10:00:00', 9876)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

| 항목 | Offset/Limit | Keyset |
|------|-------------|--------|
| 속도 | 페이지 뒤로 갈수록 느림 | 항상 일정 |
| 임의 페이지 이동 | 가능 | 불가 (앞/뒤만) |
| 새 데이터 삽입 시 | 중복/누락 발생 가능 | 안전 |
| 구현 복잡도 | 단순 | 다소 복잡 |

**무한 스크롤, 피드** → Keyset  
**페이지 번호 클릭 UI** → Offset + Covering Index Deferred Join

---

## 인덱스 설계 체크리스트

```sql
-- ORDER BY + LIMIT 패턴이면 정렬 컬럼을 인덱스 앞에
CREATE INDEX idx_orders_created ON orders (created_at DESC, id DESC);

-- WHERE + ORDER BY 함께라면 WHERE 컬럼 먼저
CREATE INDEX idx_orders_user_created ON orders (user_id, created_at DESC);
```

- 커버링 인덱스를 노리려면 `SELECT` 컬럼도 인덱스에 포함
- 인덱스 컬럼 순서: **등치(=) 조건 → 범위(<, >) → 정렬**

---

## 커버링 인덱스, 언제 쓰는가

### 왜 빠른가 — 내부 동작 이해

일반 인덱스 조회 흐름:

```
B-Tree 인덱스 탐색 → 매칭된 행마다 heap(테이블 본문) 랜덤 I/O → 결과 반환
```

커버링 인덱스 조회 흐름:

```
B-Tree 인덱스 탐색 → 인덱스 내에서 바로 결과 반환 (heap 접근 없음)
```

heap 접근은 **랜덤 I/O**다. 인덱스는 정렬된 **순차 I/O**에 가깝다.  
행 수가 많을수록, 테이블이 클수록 이 차이가 크게 벌어진다.

---

### 쓰면 좋은 상황

#### 1. 목록 조회 — SELECT 컬럼이 적고 고정적일 때

```sql
-- 게시글 목록: id, title, created_at만 필요
CREATE INDEX idx_posts_list ON posts (created_at DESC, id, title);

SELECT id, title, created_at FROM posts ORDER BY created_at DESC LIMIT 20;
-- heap 접근 없음
```

`SELECT *` 쓰는 순간 커버링 인덱스 불가. **필요한 컬럼만 SELECT하는 습관이 전제.**

#### 2. 페이지네이션 — OFFSET이 큰 경우 (Deferred Join)

이미 위에서 다룬 패턴. OFFSET이 수백~수천 이상이면 체감 효과가 크다.

#### 3. COUNT 쿼리

```sql
-- 조건에 맞는 인덱스가 있으면 heap 없이 카운트 가능
CREATE INDEX idx_orders_status ON orders (status, created_at);

SELECT COUNT(*) FROM orders WHERE status = 'PENDING';
-- EXPLAIN Extra: Using index
```

전체 카운트보다 **조건부 카운트**에서 자주 활용된다.

#### 4. 집계 + 그룹핑

```sql
CREATE INDEX idx_orders_user_status ON orders (user_id, status, amount);

SELECT user_id, status, SUM(amount)
FROM orders
GROUP BY user_id, status;
-- 인덱스 안에서 집계 완료
```

#### 5. 존재 여부 확인 (EXISTS / 단순 조건)

```sql
CREATE INDEX idx_sessions_token ON sessions (token, expires_at);

SELECT expires_at FROM sessions WHERE token = 'abc123';
-- 인증 미들웨어에서 매 요청마다 호출되는 쿼리 → 커버링 인덱스 효과 큼
```

**호출 빈도가 높을수록** 커버링 인덱스의 가성비가 올라간다.

---

### 쓰지 말아야 할 상황

#### 1. SELECT 컬럼이 너무 많을 때

```sql
-- 이건 의미없음. 인덱스가 테이블을 거의 복사하는 수준
CREATE INDEX idx_fat ON orders (id, user_id, status, amount, address, memo, created_at, updated_at);
```

인덱스도 디스크를 쓴다. 컬럼이 많아질수록 인덱스 크기 ↑, 쓰기 비용 ↑.

#### 2. 쓰기가 훨씬 많은 테이블

인덱스는 INSERT/UPDATE/DELETE마다 같이 갱신된다.  
`orders` 같은 테이블은 괜찮지만, 초당 수천 건 INSERT되는 로그성 테이블에 커버링 인덱스를 여러 개 걸면 쓰기 병목이 생긴다.

#### 3. 카디널리티가 낮은 컬럼 단독 인덱스

```sql
-- status가 'PENDING'/'DONE' 두 가지뿐이면 인덱스 효과 없음
CREATE INDEX idx_status ON orders (status);  -- 거의 의미없음
```

카디널리티가 낮으면 인덱스를 타도 대부분의 행을 읽게 된다. 커버링이고 뭐고 효과 없음.

#### 4. 테이블이 작을 때

행이 수천 건 이하면 DB가 그냥 full scan을 선택하거나 heap 비용이 무시할 수준.  
인덱스를 추가해봤자 오히려 옵티마이저 오판 리스크만 생긴다.

---

### 판단 흐름

```
이 쿼리가 자주 실행되는가?
├── 아니오 → 커버링 인덱스 필요 없음
└── 예
    │
    SELECT 컬럼 수가 적고 고정적인가?
    ├── 아니오 → Deferred Join 패턴 고려 (id만 커버링)
    └── 예
        │
        테이블 크기가 충분히 큰가? (수만 행 이상)
        ├── 아니오 → 일반 인덱스로 충분
        └── 예
            │
            테이블이 읽기 위주인가?
            ├── 아니오 → 쓰기 비용 측정 후 결정
            └── 예 → 커버링 인덱스 적용
```

---

### 실무 적용 순서

1. **EXPLAIN 먼저** — `Extra: Using index` 없으면 후보
2. **slow query log** 또는 APM에서 빈번하고 느린 쿼리 식별
3. SELECT 컬럼 최소화 (SELECT * 제거)
4. `WHERE + ORDER BY` 컬럼 조합으로 인덱스 설계
5. 필요하면 SELECT 컬럼을 인덱스에 추가 (include column)
6. EXPLAIN 재확인 → 운영 반영 전 **쓰기 부하 테스트**
