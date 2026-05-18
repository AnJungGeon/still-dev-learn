# companion object



`companion object`는 클래스 내부에 선언하는 싱글턴 객체로,  
Java의 `static` 멤버를 대체하는 Kotlin의 방식이다.

Java의 `static`, Kotlin의 `companion object` 모두 JVM의 **메서드 영역(Method Area)** 에 올라간다.  
메서드 영역은 클래스 로딩 시점에 초기화되고 프로그램 종료 시까지 유지되므로,  
인스턴스 생성 없이도 접근 가능하며, 모든 인스턴스에서 **공유할 수 있다**.

```kotlin
class MyClass {
    companion object {
        val MAX = 100
        fun create() = MyClass()
    }
}

MyClass.MAX       // 100
MyClass.create()  // MyClass 인스턴스
```

클래스당 하나만 선언할 수 있다.

---

## Java static과 비교

```java
// Java
public class Person {
    private static final String SPECIES = "Human";

    public static Person create(String name) {
        return new Person(name);
    }
}
```

```kotlin
// Kotlin
class Person(val name: String) {
    companion object {
        const val SPECIES = "Human"

        fun create(name: String) = Person(name)
    }
}

Person.SPECIES       // "Human"
Person.create("Alice")
```

Kotlin에는 `static` 키워드가 없다. `companion object` 안에 선언하면 외부에서 클래스명으로 접근할 수 있다.

---

## const val vs val

```kotlin
companion object {
    const val MAX = 100      // 컴파일 타임 상수 — 기본 타입/String만 가능
    val INSTANCE = MyClass() // 런타임 초기화 — 모든 타입 가능
}
```

- `const val` : 컴파일 시점에 값이 결정됨, Java의 `static final`과 동일
- `val` : 런타임에 초기화됨

---

## 이름 붙이기

`companion object`에 이름을 붙일 수 있다. (생략하면 `Companion`이 기본값)

```kotlin
class Person(val name: String) {
    companion object Factory {
        fun create(name: String) = Person(name)
    }
}

Person.create("Alice")          // 클래스명으로 접근
Person.Factory.create("Alice")  // 이름으로도 접근 가능
```

---

## 인터페이스 구현

`companion object`도 인터페이스를 구현할 수 있다.

```kotlin
interface Factory<T> {
    fun create(): T
}

class Person(val name: String) {
    companion object : Factory<Person> {
        override fun create() = Person("Default")
    }
}

val factory: Factory<Person> = Person  // 클래스명 자체가 Factory처럼 동작
val person = factory.create()
```

---

## object vs companion object

| | `object` | `companion object` |
|---|---|---|
| 선언 위치 | 최상위 또는 클래스 내부 | 클래스 내부만 |
| 클래스당 개수 | 여러 개 가능 | 하나만 |
| 접근 방법 | 객체명으로 직접 | 클래스명으로 |
| 용도 | 싱글턴, 유틸리티 | static 멤버 대체 |

```kotlin
// object — 독립적인 싱글턴
object Logger {
    fun log(msg: String) = println(msg)
}
Logger.log("hello")

// companion object — 클래스에 종속된 static 멤버
class Database {
    companion object {
        fun connect() = Database()
    }
}
Database.connect()
```

---

## 활용 예시

### 팩토리 메서드 패턴

생성자를 `private`으로 막고, `companion object`에서만 인스턴스를 만들 수 있게 한다.

```kotlin
class Token private constructor(val value: String) {
    companion object {
        fun of(value: String): Token? {
            if (value.isBlank()) return null
            return Token(value)
        }
    }
}

val token = Token.of("abc123")  // null 또는 Token
```

### 상수 모음

```kotlin
class HttpStatus {
    companion object {
        const val OK = 200
        const val NOT_FOUND = 404
        const val INTERNAL_SERVER_ERROR = 500
    }
}

HttpStatus.OK  // 200
```
