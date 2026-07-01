# @Transient

JPA가 해당 필드를 **DB 컬럼과 매핑하지 않도록** 제외한다.

```java
@Transient
private String tempToken;
```

---

## 언제 쓰는가

- 계산된 값, 임시 상태 등 **DB에 저장할 필요 없는 필드**
- 연관 엔티티에서 가져온 값을 캐싱하거나 가공해서 들고 있을 때
- DTO 변환 전 중간 가공용 필드

```java
@Entity
public class Order {

    private int quantity;
    private int unitPrice;

    @Transient
    private int totalPrice; // quantity * unitPrice, DB 저장 불필요

    public int getTotalPrice() {
        return quantity * unitPrice;
    }
}
```

---

## 주의: Java 표준 `transient`와 다르다

| | 역할 |
|--|------|
| `@Transient` (JPA) | JPA 매핑에서 제외. 직렬화에는 포함됨 |
| `transient` (Java 키워드) | 직렬화(`Serializable`)에서 제외. JPA는 여전히 매핑 시도 |

JPA에서 제외하려면 반드시 `@Transient` 어노테이션을 써야 한다.  
Java `transient` 키워드로는 JPA 매핑이 제외되지 않는다.

---

## 패키지 주의

```java
import javax.persistence.Transient;   // JPA (Jakarta EE 8 이하)
import jakarta.persistence.Transient; // JPA (Jakarta EE 9+, Spring Boot 3.x)
```

`java.beans.Transient`도 존재하는데 이건 JPA와 무관하다. import 확인 필수.
