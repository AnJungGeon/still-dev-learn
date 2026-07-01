# @AttributeOverride / @Convert

BaseEntity 컬럼을 기존 DB 스키마에 맞게 재정의할 때 사용한 조합.

## 배경

기존 테이블에 이미 `indate` 같은 레거시 컬럼명이 존재하는 상황에서  
공통 BaseEntity(`createdAt`)를 상속받아 쓰고 싶을 때 문제가 생긴다.

```
BaseEntity.createdAt  →  DB 컬럼명 기본값: created_at
실제 테이블 컬럼명: indate  →  불일치
```

---

## @AttributeOverride

상속받은 또는 임베디드된 컬럼의 **이름/정의를 재정의**한다.

```java
@AttributeOverride(name = "createdAt", column = @Column(name = "indate", updatable = false))
```

| 속성 | 설명 |
|------|------|
| `name` | 부모 엔티티/임베디드 클래스의 **필드명** |
| `column` | 이 엔티티에서 사용할 실제 컬럼 정의 |

- 여러 개 재정의 시 `@AttributeOverrides({ ... })`로 감쌈
- `@MappedSuperclass` 또는 `@Embeddable` 상속 관계에서만 의미 있음

---

## @Convert

특정 필드에 **AttributeConverter를 적용**한다.  
DB ↔ Java 타입 변환 (예: `String` ↔ `LocalDateTime`, `String` ↔ `Enum`)

```java
@Convert(attributeName = "createdAt", converter = CreatedAtConverter.class)
```

| 속성 | 설명 |
|------|------|
| `converter` | 적용할 `AttributeConverter` 구현체 |
| `attributeName` | 임베디드/상속 필드에 적용할 때 필드명 지정 |
| `disableConversion` | true면 해당 필드 컨버터 비활성화 (기본 false) |

`attributeName`은 일반 단일 필드에 직접 붙일 때는 생략해도 된다.  
**상속받은 필드**에 적용할 때는 명시해야 한다.

### AttributeConverter 구현 예시

```java
@Converter
public class CreatedAtConverter implements AttributeConverter<LocalDateTime, String> {

    @Override
    public String convertToDatabaseColumn(LocalDateTime attribute) {
        // LocalDateTime → DB 저장값 (예: "20240115103000")
        return attribute.format(DateTimeFormatter.ofPattern("yyyyMMddHHmmss"));
    }

    @Override
    public LocalDateTime convertToEntityAttribute(String dbData) {
        // DB 저장값 → LocalDateTime
        return LocalDateTime.parse(dbData, DateTimeFormatter.ofPattern("yyyyMMddHHmmss"));
    }
}
```

---

## 두 어노테이션을 함께 쓰는 이유

```java
@AttributeOverride(name = "createdAt", column = @Column(name = "indate", updatable = false))
@Convert(attributeName = "createdAt", converter = CreatedAtConverter.class)
public class Article extends BaseEntity { ... }
```

각자 하는 일이 다르다.

| | 역할 |
|--|------|
| `@AttributeOverride` | **컬럼명** 재정의 (`createdAt` → `indate`) |
| `@Convert` | **타입 변환** 재정의 (DB 포맷 ↔ Java 타입) |

`@AttributeOverride`만 쓰면 컬럼명은 맞지만 타입 변환이 안 된다.  
`@Convert`만 쓰면 변환은 되지만 컬럼명이 여전히 `created_at`으로 매핑된다.

---

## indate 필드 직접 선언하면 안 되는 이유

`@AttributeOverride`로 `createdAt → indate`를 매핑한 상태에서  
엔티티에 `indate` 필드를 **별도로 선언하면 Hibernate가 충돌**을 낸다.

```java
// 잘못된 예
@AttributeOverride(name = "createdAt", column = @Column(name = "indate"))
public class Article extends BaseEntity {

    @Column(name = "indate")
    private String indate; // ← 같은 컬럼을 두 번 매핑 → 충돌
}
```

```
org.hibernate.MappingException: Repeated column in mapping for entity
```

**해결:** `indate` 필드를 따로 선언하지 않는다.  
상속받은 `createdAt`이 `indate` 컬럼을 담당하므로 `createdAt`으로만 접근하면 된다.

```java
// 올바른 예
@AttributeOverride(name = "createdAt", column = @Column(name = "indate", updatable = false))
@Convert(attributeName = "createdAt", converter = CreatedAtConverter.class)
public class Article extends BaseEntity {
    // indate 필드 없음. article.getCreatedAt()으로 접근
}
```

---

## 클래스 레벨 vs 필드 레벨

```java
// 클래스 레벨: 상속 필드에 적용할 때
@AttributeOverride(name = "createdAt", column = @Column(name = "indate"))
@Convert(attributeName = "createdAt", converter = SomeConverter.class)
public class Article extends BaseEntity { ... }

// 필드 레벨: 직접 선언한 필드에 적용할 때
@Convert(converter = SomeConverter.class)
private LocalDateTime createdAt;
```

상속받은 필드는 해당 클래스 안에 코드가 없으므로 반드시 **클래스 레벨**에 붙여야 한다.
