# data class


`data class`는 데이터를 담는 것이 주목적인 클래스를 선언할 때 사용한다.  
컴파일러가 `equals()`, `hashCode()`, `toString()`, `copy()`, `componentN()` 함수를 자동으로 생성해준다.

```kotlin
data class Person(val name: String, val age: Int)
```

Java의 Lombok `@Data`, record(Java 16+)와 유사하지만 더 강력하다.

---

## 자동 생성 함수

### toString()

```kotlin
data class Person(val name: String, val age: Int)

val p = Person("Alice", 30)
println(p)  // Person(name=Alice, age=30)
```

일반 클래스라면 `Person@1a2b3c` 같은 주소값이 출력되지만,  
`data class`는 프로퍼티 값을 포함한 읽기 좋은 문자열을 반환한다.

---

### equals() / hashCode()

```kotlin
val p1 = Person("Alice", 30)
val p2 = Person("Alice", 30)

println(p1 == p2)   // true  (값 비교)
println(p1 === p2)  // false (참조 비교)
```

- `==` : `equals()` 호출 → 프로퍼티 값이 같으면 `true`
- `===` : 참조(주소) 비교 → 다른 객체이므로 `false`

일반 클래스에서는 `==`도 참조 비교가 되어 `false`가 나온다.

---

### copy()

객체의 일부 프로퍼티만 바꾼 새 객체를 생성할 때 사용한다.

```kotlin
val alice = Person("Alice", 30)
val olderAlice = alice.copy(age = 31)

println(alice)       // Person(name=Alice, age=30)
println(olderAlice)  // Person(name=Alice, age=31)
```

불변 객체(`val`)를 유지하면서 변경된 버전을 쉽게 만들 수 있다.

---

### componentN() — 구조 분해 선언

프로퍼티 순서대로 `component1()`, `component2()` ... 함수가 생성된다.  
이를 통해 구조 분해 선언(destructuring declaration)이 가능하다.

```kotlin
val (name, age) = Person("Alice", 30)
println(name)  // Alice
println(age)   // 30
```

`for` 루프에서도 활용 가능하다.

```kotlin
val people = listOf(Person("Alice", 30), Person("Bob", 25))

for ((name, age) in people) {
    println("$name is $age years old")
}
// Alice is 30 years old
// Bob is 25 years old
```

---

## 주의사항

### 주 생성자에 프로퍼티가 있어야 함

자동 생성 함수들은 **주 생성자에 선언된 프로퍼티**만 대상으로 한다.  
클래스 본문에 선언한 프로퍼티는 포함되지 않는다.

```kotlin
data class Person(val name: String) {
    var nickname: String = ""  // equals/hashCode/toString에 포함되지 않음
}

val p1 = Person("Alice").apply { nickname = "Al" }
val p2 = Person("Alice").apply { nickname = "Ally" }

println(p1 == p2)  // true — nickname은 비교 대상이 아님
```

### val vs var

주 생성자 파라미터는 `val`(불변)을 권장한다.  
`var`로 선언하면 `copy()`의 불변성 장점을 살리기 어렵고,  
`hashCode`가 변하면 `HashMap`/`HashSet`에서 버그가 생긴다.

```kotlin
// 권장
data class Point(val x: Int, val y: Int)

// 비권장
data class Point(var x: Int, var y: Int)
```

### 상속 불가

`data class`는 다른 클래스를 상속할 수 없다. (인터페이스 구현은 가능)

```kotlin
// 컴파일 에러
data class Employee(val name: String) : Person("name")

// 가능
data class Employee(val name: String) : Serializable
```

---

## Java와 비교

```java
// Java + Lombok
@Data
public class Person {
    private final String name;
    private final int age;
}
```

```kotlin
// Kotlin
data class Person(val name: String, val age: Int)
```

Lombok `@Data`는 `@Getter`, `@Setter`, `@EqualsAndHashCode`, `@ToString`, `@RequiredArgsConstructor`를 합친 어노테이션이다.  
Kotlin `data class`와 거의 동일한 수준이지만 차이점이 있다.

| 기능 | Lombok `@Data` | Kotlin `data class` |
|---|---|---|
| `toString()` | O | O |
| `equals()` / `hashCode()` | O | O |
| getter | O | O (프로퍼티로 자동 제공) |
| setter | O (`val`이 없으므로 기본 생성) | X (`val` 사용 시 불변) |
| `copy()` | X (별도 `@Builder` 필요) | O |
| 구조 분해 선언 | X | O |
| 빌드 도구 의존성 | O (Lombok 플러그인 필요) | X |

---

## 실전 활용 예시

### API 응답 모델

```kotlin
data class UserResponse(
    val id: Long,
    val name: String,
    val email: String
)
```

### 상태 관리 (불변 업데이트)

```kotlin
data class UiState(
    val isLoading: Boolean = false,
    val data: List<String> = emptyList(),
    val error: String? = null
)

val state = UiState()
val loadingState = state.copy(isLoading = true)
val successState = loadingState.copy(isLoading = false, data = listOf("A", "B"))
```

### Map의 키로 사용

`equals()`와 `hashCode()`가 올바르게 구현되어 있어 `Map` 키로 안전하게 사용 가능하다.

```kotlin
data class Coordinate(val x: Int, val y: Int)

val map = mutableMapOf<Coordinate, String>()
map[Coordinate(0, 0)] = "origin"

println(map[Coordinate(0, 0)])  // origin
```
