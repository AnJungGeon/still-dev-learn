# `@Positive` 어노테이션

숫자 필드/파라미터가 **0보다 큰 값**인지 검증하는 Bean Validation 어노테이션.

`jakarta.validation.constraints.Positive` (Spring Boot 3+) / `javax.validation.constraints.Positive` (Spring Boot 2 이하)

---

## 사용 예시

```java
public class OrderRequest {
    @Positive
    private int quantity;

    @Positive
    private BigDecimal price;
}
```

```java
@PostMapping("/orders")
public void createOrder(@Valid @RequestBody OrderRequest request) {
    // quantity, price가 0 이하이면 여기 도달 전에 400 에러
}
```

컨트롤러 파라미터에 바로 붙일 때는 클래스에 `@Validated`가 있어야 동작한다.

```java
@Validated
@RestController
public class OrderController {

    @GetMapping("/orders/{id}")
    public Order getOrder(@Positive @PathVariable Long id) { ... }
}
```

---

## 관련 어노테이션 비교

| 어노테이션 | 조건 |
|------|------|
| `@Positive` | `x > 0` |
| `@PositiveOrZero` | `x >= 0` |
| `@Negative` | `x < 0` |
| `@NegativeOrZero` | `x <= 0` |
| `@Min(1)` | `x >= 1` (경계값 임의 지정 가능) |

`id`, `quantity`, `count`처럼 "0 또는 음수면 의미 없는 값"에는 `@Positive`, 재고 수량처럼 0을 허용해야 하면 `@PositiveOrZero`.

---

## 적용 대상

`int`, `long`, `double` 등 기본 숫자 타입과 `BigDecimal`, `BigInteger` 등 래퍼/숫자 타입에 적용 가능. `null`은 유효한 것으로 간주하고 통과시키므로 (검증 스킵), null 불가 조건이 필요하면 `@NotNull`을 같이 붙여야 한다.

```java
@NotNull
@Positive
private Integer quantity;
```

---

## 주의사항

- 메서드 파라미터에 붙이려면 클래스에 `@Validated` 필요 (`@Valid`는 객체 필드 검증용, 파라미터 단독 검증엔 안 먹힘)
- `null`을 걸러내지 않으므로 `@NotNull`과 조합해서 쓰는 경우가 많음
- 검증 실패 시 `ConstraintViolationException`(파라미터) 또는 `MethodArgumentNotValidException`(`@RequestBody`)이 발생 — 둘 다 잡아서 400으로 변환하는 공통 `@ExceptionHandler` 필요
