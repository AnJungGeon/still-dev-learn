# @DynamicInsert / @DynamicUpdate

Hibernate 전용 어노테이션(`org.hibernate.annotations`). JPA 표준은 아니다.

기본적으로 Hibernate는 엔티티당 INSERT/UPDATE SQL을 **애플리케이션 구동 시점에 미리 생성**해두고 재사용한다.  
이때 값이 `null`이든 변경이 없든 **모든 컬럼을 포함**시킨다.

```kotlin
@Entity
class Member(
    var name: String,
    var email: String? = null,
    var status: String = "ACTIVE"
)
```

```sql
-- 기본 동작: email이 null이어도 포함됨
insert into member (name, email, status) values (?, ?, ?)
update member set name=?, email=?, status=? where id=?
```

---

## @DynamicInsert

INSERT 시 **null인 필드를 제외**하고 SQL을 생성한다.

```kotlin
@Entity
@DynamicInsert
class Member(
    var name: String,
    var email: String? = null,
    var status: String = "ACTIVE"
)
```

```sql
-- email이 null이면 컬럼 자체가 빠짐 → DB의 DEFAULT 값이 적용됨
insert into member (name, status) values (?, ?)
```

### 언제 쓰는가
- 컬럼에 `DEFAULT` 값이 걸려 있고, null일 때 그 기본값을 그대로 쓰고 싶을 때
- nullable 컬럼이 많은 넓은 테이블

---

## @DynamicUpdate

UPDATE 시 **실제로 변경된 필드만** SQL에 포함한다.

```kotlin
@Entity
@DynamicUpdate
class Member(
    var name: String,
    var email: String? = null,
    var status: String = "ACTIVE"
)
```

```kotlin
member.status = "INACTIVE" // status만 변경
```

```sql
-- name, email은 안 바뀌었으므로 SQL에서 제외
update member set status=? where id=?
```

### 언제 쓰는가
- 컬럼이 많은 엔티티에서 일부 필드만 자주 바뀔 때 (불필요한 컬럼 쓰기 감소)
- 동시에 여러 트랜잭션이 **서로 다른 컬럼**을 수정할 때, 변경하지 않은 컬럼까지 덮어써서 생기는 데이터 유실 방지
- DB 트리거가 특정 컬럼 값을 갱신하는데, 그 컬럼을 건드리지 않는 UPDATE에서까지 매번 스쳐 지나가듯 SET하고 싶지 않을 때

---

## 트레이드오프

- 기본 방식은 SQL을 시작 시점에 캐싱해서 재사용 → 빠름
- `@DynamicInsert`/`@DynamicUpdate`는 **매번 런타임에 SQL을 동적으로 생성** → 약간의 오버헤드 발생
- 컬럼 수가 적은 테이블에서는 이득보다 오버헤드가 더 클 수 있으니, 넓은 테이블·nullable이 많은 경우에 우선 고려

---

## 함께 쓰이는 경우

두 어노테이션은 함께 붙이는 경우가 많다.

```kotlin
@Entity
@DynamicInsert
@DynamicUpdate
class Member(
    var name: String,
    var email: String? = null,
    var status: String = "ACTIVE"
)
```
