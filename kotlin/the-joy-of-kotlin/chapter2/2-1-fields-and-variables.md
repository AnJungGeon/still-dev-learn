# 2-1. 필드와 변수

## Java와의 차이점

### val / var

- `val` — 불변(immutable), Java의 `final`과 유사
- `var` — 가변(mutable), 재할당 가능
- 함수형 프로그래밍에서는 `val` 사용을 권장

```kotlin
val name = "Kotlin"   // 재할당 불가
var count = 0         // 재할당 가능
```

### 타입 추론

- Java는 타입을 명시해야 하지만, 코틀린은 타입을 추론함
- 명시도 가능

```kotlin
val name: String = "Kotlin"  // 명시
val name = "Kotlin"          // 추론
```

 **JS의 타입 추론과 비교**
> - JS(또는 TypeScript 없이)는 런타임에 타입이 결정되는 **동적 타입**
> - Kotlin은 컴파일 타임에 타입을 추론하는 **정적 타입**
> - 겉보기엔 비슷하지만 Kotlin은 타입 안전성이 컴파일 단계에서 보장됨

```kotlin
val name = "Kotlin"  // String으로 추론, 이후 다른 타입 대입 시 컴파일 에러
```

## 2-1.2 가변 필드 사용하기

- `var`로 선언한 프로퍼티는 재할당 가능
- 함수형 프로그래밍에서는 가변 상태는 버그의 원인이 되므로 최대한 `val` 사용을 권장
- 불가피하게 가변이 필요한 경우에만 `var` 사용

```kotlin
var count = 0
count = 1  // 가능

val name = "Kotlin"
name = "Java"  // 컴파일 에러
```

> **Java와의 차이점 — null**
> - Java는 모든 참조 타입에 기본적으로 `null` 허용 → NPE(NullPointerException) 위험
> - Kotlin은 기본적으로 `null` 불가, `null`을 허용하려면 `?`를 명시해야 함

```kotlin
var name: String = null   // 컴파일 에러
var name: String? = null  // 가능 (nullable)
```

## 2-1.3 지연 초기화 이해하기

- Kotlin은 프로퍼티를 선언할 때 반드시 초기화해야 함
- 하지만 초기값을 나중에 할당해야 하는 경우, 두 가지 방법을 제공

### lateinit

- `var`에만 사용 가능
- 나중에 초기화할 것임을 명시, `null` 없이 non-null 타입 유지
- 초기화 전에 접근하면 런타임 예외 발생

```kotlin
lateinit var name: String
name = "Kotlin"  // 나중에 초기화
```

### lazy

- `val`에만 사용 가능
- 처음 접근하는 순간에 초기화 (지연 초기화)
- 람다로 초기화 로직을 정의

```kotlin
val name: String by lazy {
    "Kotlin"  // 처음 name에 접근할 때 실행
}
```

> **Java와의 차이점**
> - Java는 선언 시 초기화하지 않으면 기본값(`null`, `0` 등)으로 자동 설정
> - Kotlin은 명시적 초기화를 강제하여 null 관련 버그를 컴파일 단계에서 방지

### lazy 주요 사용 사례

- 무거운 객체 초기화 — DB 연결, 대용량 설정 파일 파싱 등 비용이 큰 작업을 실제로 필요할 때까지 미룰 때
- 순환 의존성 — A가 B를 참조하고 B가 A를 참조하는 경우 한쪽을 `lazy`로 해결
- 자주 안 쓰는 기능 — 항상 필요하지 않은 리소스를 미리 만들지 않으려 할 때

> 스프링에서는 빈 관리를 프레임워크가 해주기 때문에 `lazy`를 쓸 일이 거의 없음
> 굳이 쓴다면 설정값을 처음 접근할 때 읽어오는 경우 정도이며, 보통은 `@Bean`이나 `@Value`로 대체

```kotlin
val config: AppConfig by lazy {
    loadConfig()  // 처음 접근할 때 한 번만 로드
}
```

### lateinit 주요 사용 사례

- 스프링에서 `@Autowired`로 필드 주입을 받는 경우
- 테스트 코드에서 `@BeforeEach`로 나중에 세팅하는 경우

```kotlin
// 필드 주입 (권장하지 않음)
@Autowired
lateinit var userService: UserService

// 테스트 코드
class UserServiceTest {
    @Autowired
    lateinit var userService: UserService
}
```

> 스프링에서는 **생성자 주입**이 권장 방식이므로 `lateinit`을 쓸 일이 많지 않음
>
> ```kotlin
> @Service
> class UserController(
>     private val userService: UserService  // 생성자 주입 → lateinit 불필요
> )
> ```

### 필드가 아닌 프로퍼티

- Java는 필드(field) + getter/setter를 직접 작성
- 코틀린은 프로퍼티(property)로 getter/setter를 자동 제공

```kotlin
// Java
private String name;
public String getName() { return name; }
public void setName(String name) { this.name = name; }

// Kotlin
var name: String = "Kotlin"  // getter/setter 자동 생성
val name: String = "Kotlin"  // getter만 자동 생성
```

> **Java Lombok과 비교**
> - Java에서는 `@Getter`, `@Setter` 등 Lombok 어노테이션으로 보일러플레이트를 줄임
> - Kotlin은 별도 라이브러리 없이 언어 자체에서 프로퍼티를 기본 제공
> - Lombok 의존성이 필요 없음
