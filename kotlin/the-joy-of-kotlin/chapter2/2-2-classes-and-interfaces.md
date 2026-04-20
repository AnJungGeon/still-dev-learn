# 2-2. 클래스와 인터페이스

코틀린 클래스는 자바와 상당히 다른 구문을 사용한다.

`String` 타입의 `name` 프로퍼티를 가진 `Person` 클래스를 코틀린으로 정의하면:

**Kotlin**
```kotlin
class Person constructor(name: String) {
    val name: String

    init {
        this.name = name
    }
}
```

**Java**
```java
public final class Person {

    private final String name;

    public Person(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }
}
```

## 접근 제한자

- 코틀린 클래스는 기본적으로 `public`
- 공개하지 않으려면 접근 제한자를 명시해야 함 (`private`, `internal` 등)

## 상속

- 코틀린 클래스는 기본적으로 상속 불가 (`final`)
- `open` — "확장에 열려있다"는 의미로, 상속을 허용하려면 명시해야 함

```kotlin
class Animal          // 상속 불가
open class Animal     // 상속 가능
```

## 파일명과 클래스명

- Java는 `public` 클래스명과 파일명이 반드시 같아야 함
- Kotlin은 달라도 되고, 한 파일에 여러 클래스를 넣어도 됨
- 단, 가독성을 위해 파일명과 클래스명을 맞춰주는 것을 권장

## 2-2.1 예제 코드 간결하게 만들기

앞선 `Person` 클래스를 단계적으로 간결하게 줄일 수 있다.

**1단계 — `constructor` 키워드 생략 가능**
```kotlin
class Person(name: String) {
    val name: String

    init {
        this.name = name
    }
}
```

**2단계 — 생성자 파라미터에 `val`/`var`를 붙이면 프로퍼티 선언과 초기화를 동시에**
```kotlin
class Person(val name: String)
```

> Java에서 생성자, 필드, getter를 따로 작성하던 것을 한 줄로 표현 가능 (Lombok 같은 어노테이션 없이도 가능)

## 2-2.2 인터페이스를 구현하거나 클래스를 확장하기

- Java의 `implements`, `extends` 대신 `:` 하나로 통일

```kotlin
// 인터페이스 구현
class Person(val name: String) : Serializable

// 클래스 상속
open class Animal
class Dog : Animal()

// 둘 다
class Dog : Animal(), Runnable
```

상속할 클래스는 `open`이어야 하며, 부모 클래스 이름 뒤에 반드시 괄호`()`를 붙여야 함 (부모 생성자 호출)
인터페이스는 `()` 불필요

```kotlin
open class Animal(val name: String)
class Dog(name: String) : Animal(name)  // 부모 생성자에 인자 전달
```

## 2-2.3 클래스 인스턴스화하기

- Java와 달리 `new` 키워드 없이 인스턴스화

```kotlin
// Java
Person person = new Person("Kotlin");

// Kotlin
val person = Person("Kotlin")
```

## 2-2.4 프로퍼티 생성자 오버로드하기

- Kotlin은 **기본값(default parameter)**으로 생성자 오버로드를 대체

```kotlin
// Java — 오버로드 필요
public Person(String name) {
    this(name, 0);
}

public Person(String name, int age) {
    this.name = name;
    this.age = age;
}

// Kotlin — 기본값으로 해결
class Person(val name: String, val age: Int = 0)

val p1 = Person("Kotlin")       // age = 0
val p2 = Person("Kotlin", 10)   // age = 10
```

- 부 생성자(secondary constructor)가 필요하면 `constructor` 키워드로 추가 가능

```kotlin
class Person(val name: String) {
    constructor(name: String, age: Int) : this(name) {
        // 추가 로직
    }
}
```

> 기본값 방식이 더 간결하므로 부 생성자는 잘 쓰지 않음