# QueryDSL — Projections 정리

QueryDSL에서 엔티티 전체가 아닌 **원하는 컬럼만 DTO로 조회**할 때 사용하는 두 가지 방식.

---

## 목차

- [Projections.constructor](#projectionsConstructor)
- [@QueryProjection](#queryprojection)
- [비교 정리](#비교-정리)

---

## Projections.constructor

DTO 생성자를 **런타임에 리플렉션으로 호출**하는 방식.  
DTO에 QueryDSL 의존성이 없어서 순수 Java 객체로 유지할 수 있다.

```java
// DTO (QueryDSL 의존 없음)
public record MemberDto(String name, int age) {}
```

```java
List<MemberDto> result = queryFactory
    .select(Projections.constructor(MemberDto.class,
        member.name,
        member.age
    ))
    .from(member)
    .fetch();
```

**주의 사항**

- 생성자 파라미터 **순서와 타입**이 정확히 일치해야 한다.
- 잘못된 타입이나 순서는 **런타임 오류**로만 잡힌다 — 컴파일 타임에 검출 불가.

```java
// 컴파일은 되지만 런타임에 오류 발생
Projections.constructor(MemberDto.class,
    member.age,   // int
    member.name   // String  → 순서가 바뀌어도 컴파일러는 모름
)
```

---

## @QueryProjection

DTO 생성자에 `@QueryProjection`을 붙이면 **Q타입 생성자**가 함께 생성된다.  
Q타입을 통해 **컴파일 타임에 타입 안정성**을 확보할 수 있다.

**1. DTO 생성자에 애노테이션 추가**

```java
import com.querydsl.core.annotations.QueryProjection;

public class MemberDto {
    private String name;
    private int age;

    @QueryProjection  // Q타입 생성자 생성 대상으로 지정
    public MemberDto(String name, int age) {
        this.name = name;
        this.age = age;
    }
}
```

**2. Q타입 생성 (Gradle)**

```bash
./gradlew compileQuerydsl
# 또는
./gradlew build
```

생성 결과:

```java
// QMemberDto (자동 생성)
public class QMemberDto extends ConstructorExpression<MemberDto> {
    public QMemberDto(Expression<String> name, Expression<Integer> age) { ... }
}
```

**3. 쿼리에서 사용**

```java
List<MemberDto> result = queryFactory
    .select(new QMemberDto(
        member.name,
        member.age
    ))
    .from(member)
    .fetch();
```

- 파라미터 타입이 맞지 않으면 **컴파일 오류**로 즉시 검출된다.
- IDE 자동완성도 지원된다.

---

## 비교 정리

| 항목 | `Projections.constructor` | `@QueryProjection` |
|---|---|---|
| **타입 안정성** | ❌ 런타임 오류 | ✅ 컴파일 오류 |
| **DTO 의존성** | QueryDSL 의존 없음 | `@QueryProjection` 애노테이션 필요 |
| **Q타입 생성** | 불필요 | 필요 (`compileQuerydsl`) |
| **IDE 자동완성** | 미지원 | 지원 |
| **사용 편의성** | 단순, 설정 최소 | 초기 설정 후 안전하고 편함 |

**선택 기준**

```
DTO를 QueryDSL에 의존시키기 싫다     → Projections.constructor
컴파일 타임 안전성이 중요하다         → @QueryProjection
```

> DTO가 QueryDSL 레이어에만 쓰인다면 `@QueryProjection`,  
> 여러 모듈에서 공유되는 공통 DTO라면 `Projections.constructor`를 쓰는 경우가 많다.
