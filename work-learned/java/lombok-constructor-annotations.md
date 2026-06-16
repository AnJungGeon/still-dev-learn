# Lombok 생성자 어노테이션 비교

`@NoArgsConstructor`, `@AllArgsConstructor`, `@RequiredArgsConstructor` 비교.

---

## 비교표

| 어노테이션 | 생성자 파라미터 |
|------|------|
| `@NoArgsConstructor` | 없음 (기본 생성자) |
| `@AllArgsConstructor` | 모든 필드 |
| `@RequiredArgsConstructor` | `final` 필드 + `@NonNull` 필드만 |

---

## 예시

```java
public class User {
    private final Long id;
    @NonNull
    private String name;
    private int age;
}
```

### `@NoArgsConstructor`

```java
public User() {}
```

JPA 엔티티는 프록시 생성을 위해 기본 생성자가 필요해서 거의 필수로 붙는다.

```java
@NoArgsConstructor
@Entity
public class User {
    @Id
    private Long id;
    private String name;
}
```

---

### `@AllArgsConstructor`

```java
public User(Long id, String name, int age) {
    this.id = id;
    this.name = name;
    this.age = age;
}
```

필드 전체를 받는 생성자. 필드 순서가 그대로 파라미터 순서가 되므로 필드가 많아지면 가독성이 떨어진다.

---

### `@RequiredArgsConstructor`

```java
public User(Long id, String name) {
    this.id = id;
    this.name = name;
}
// age는 int 기본값 0
```

`final` 필드와 `@NonNull` 필드만 파라미터로 받는다. **의존성 주입(DI)**에서 가장 많이 쓰인다.

```java
@RequiredArgsConstructor
@Service
public class UserService {
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    // 생성자 주입 코드 작성 없이 자동 생성됨
}
```

---

## 실무 사용 기준

| 상황 | 어노테이션 |
|------|------|
| Spring `@Service`, `@Component` 등 생성자 주입 | `@RequiredArgsConstructor` |
| JPA `@Entity` (프록시/리플렉션용) | `@NoArgsConstructor` |
| 테스트 코드에서 객체 빠르게 생성 | `@AllArgsConstructor` |

세 개를 한 클래스에 같이 쓰는 경우도 있다 (Entity + 빌더 조합 등).

```java
@NoArgsConstructor              // JPA용 기본 생성자
@AllArgsConstructor(access = AccessLevel.PRIVATE)  // 빌더 내부용
@Builder
@Entity
public class User { ... }
```

---

## 주의사항

- `@RequiredArgsConstructor`는 필드 선언 순서가 바뀌면 생성자 파라미터 순서도 바뀜 — 리플렉션 기반 호출 시 주의
- `final` 필드가 하나도 없으면 `@RequiredArgsConstructor`는 빈 생성자와 동일 (`@NoArgsConstructor`와 같은 효과)
- `@NonNull` 필드는 생성자에서 null 체크가 자동으로 들어감 (`NullPointerException` 발생)
