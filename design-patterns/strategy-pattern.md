# 전략 패턴 (Strategy Pattern)

GoF 디자인 패턴 중 하나로, **알고리즘 군을 정의하고 각각을 캡슐화해서 교환 가능하게** 만드는 패턴이다.  
클라이언트로부터 알고리즘을 분리해 독립적으로 변경할 수 있다.

---

## 3가지 설계 원칙

### 1. 변하는 부분을 캡슐화하라
> *Encapsulate what varies*

바뀌는 알고리즘(행동)을 별도 클래스로 분리해서 나머지 코드에 영향을 주지 않도록 격리한다.

```java
interface FlyBehavior {
    void fly();
}

class FlyWithWings implements FlyBehavior {
    public void fly() { System.out.println("날개로 날기"); }
}

class NoFly implements FlyBehavior {
    public void fly() { System.out.println("못 날아요"); }
}
```

---

### 2. 구현이 아닌 인터페이스에 맞춰 프로그래밍하라
> *Program to an interface, not an implementation*

구체 클래스가 아닌 인터페이스 타입으로 참조해서 구현체에 대한 의존을 제거한다.

```java
class Duck {
    private FlyBehavior flyBehavior;  // FlyWithWings가 아닌 FlyBehavior에 의존

    public Duck(FlyBehavior flyBehavior) {
        this.flyBehavior = flyBehavior;
    }

    public void performFly() {
        flyBehavior.fly();
    }
}
```

---

### 3. 상속보다 구성(Composition)을 활용하라
> *Favor composition over inheritance*

행동을 상속받는 대신 행동 객체를 **필드로 포함**시켜 런타임에 교체 가능하게 만든다.

```java
Duck duck = new Duck(new FlyWithWings());
duck.performFly();  // "날개로 날기"

duck.flyBehavior = new NoFly();  // 런타임 교체
duck.performFly();  // "못 날아요"
```

---

## 구성 요소

| 역할 | 설명 |
|------|------|
| Strategy 인터페이스 (전략 인터페이스) | 알고리즘의 공통 인터페이스 정의 |
| Concrete Strategy (구체 전략) | 인터페이스를 구현한 실제 알고리즘 |
| Context (문맥) | Strategy를 필드로 가지고 위임해서 실행 |
| Client (클라이언트) | 어떤 전략을 쓸지 결정하는 주체 |

---

## 실제 웹 개발 예시 (Strategy + Factory)

실무에서는 순수한 GoF 전략 패턴보다 **전략 패턴 + 팩토리 패턴** 조합으로 주로 쓴다.  
클라이언트가 `enum` 타입을 넘기면, 팩토리가 해당 전략 구현체를 선택해 실행하는 방식이다.

---

### 시나리오

Email, SMS, Push 알림은 테이블 구조가 동일하다. (`id`, `title`, `content`, `recipient`, `status`, `created_at`)  
타입 하나(`NotificationType`)로 처리해서 컨트롤러와 서비스의 중복 코드를 제거한다.

---

### 클라이언트 (Client, JS)

```javascript
// 어떤 전략을 쓸지 클라이언트가 결정
fetch('/api/notifications', {
  method: 'POST',
  body: JSON.stringify({ type: 'EMAIL', recipient: 'user@example.com', title: '주문 완료' })
});
```

---

### NotificationType (열거형, enum)

```java
public enum NotificationType {
    EMAIL, SMS, PUSH
}
```

---

### Strategy 인터페이스 (전략 인터페이스)

```java
public interface NotificationRepository {
    void save(String recipient, String title, String content);
    List<String> findAllByRecipient(String recipient);
}
```

---

### Concrete Strategy (구체 전략)

```java
@Repository
public class EmailNotificationRepository implements NotificationRepository {
    public void save(String recipient, String title, String content) {
        // email_notification 테이블에 저장
    }
    public List<String> findAllByRecipient(String recipient) {
        return List.of("주문 완료", "배송 시작");
    }
}

@Repository
public class SmsNotificationRepository implements NotificationRepository {
    public void save(String recipient, String title, String content) {
        // sms_notification 테이블에 저장
    }
    public List<String> findAllByRecipient(String recipient) {
        return List.of("인증번호 발송");
    }
}

@Repository
public class PushNotificationRepository implements NotificationRepository {
    public void save(String recipient, String title, String content) {
        // push_notification 테이블에 저장
    }
    public List<String> findAllByRecipient(String recipient) {
        return List.of("새 메시지 도착");
    }
}
```

---

### 팩토리 (Factory, 전략 선택)

전략 선택 책임을 한 곳에 모아두어 서비스 여러 곳에 선택 로직이 흩어지지 않게 한다.

```java
@Component
public class NotificationRepositoryFactory {

    private final EmailNotificationRepository emailRepo;
    private final SmsNotificationRepository smsRepo;
    private final PushNotificationRepository pushRepo;

    public NotificationRepositoryFactory(
            EmailNotificationRepository emailRepo,
            SmsNotificationRepository smsRepo,
            PushNotificationRepository pushRepo) {
        this.emailRepo = emailRepo;
        this.smsRepo = smsRepo;
        this.pushRepo = pushRepo;
    }

    public NotificationRepository repo(NotificationType type) {
        return switch (type) {
            case EMAIL -> emailRepo;
            case SMS   -> smsRepo;
            case PUSH  -> pushRepo;
        };
    }
}
```

---

### 서비스 (Service, Context 문맥)

```java
@Service
public class NotificationService {

    private final NotificationRepositoryFactory factory;

    public NotificationService(NotificationRepositoryFactory factory) {
        this.factory = factory;
    }

    public void send(NotificationType type, String recipient, String title, String content) {
        factory.repo(type).save(recipient, title, content);
    }

    public List<String> findAll(NotificationType type, String recipient) {
        return factory.repo(type).findAllByRecipient(recipient);
    }
}
```

타입이 추가되어도 서비스 코드는 변경되지 않는다.  
`NotificationType`에 값 추가 → Repository 구현체 추가 → Factory 분기 추가 → 끝.

---

### 컨트롤러 (Controller)

```java
@RestController
@RequestMapping("/api/notifications")
public class NotificationController {

    private final NotificationService notificationService;

    public NotificationController(NotificationService notificationService) {
        this.notificationService = notificationService;
    }

    @PostMapping
    public ResponseEntity<Void> send(@RequestBody NotificationRequest request) {
        notificationService.send(request.type(), request.recipient(), request.title(), request.content());
        return ResponseEntity.ok().build();
    }

    @GetMapping
    public ResponseEntity<List<String>> findAll(
            @RequestParam NotificationType type,
            @RequestParam String recipient) {
        return ResponseEntity.ok(notificationService.findAll(type, recipient));
    }
}
```

---

## 핵심 정리

| | 순수 GoF | 실무 (Strategy + Factory) |
|---|---|---|
| 전략 결정 시점 | Context 생성 시 or 런타임 주입 | 요청마다 enum으로 동적 선택 |
| 전략 선택 위치 | Client가 직접 Context에 주입 | Factory가 enum 보고 선택 |
| 확장 방법 | 새 Concrete Strategy 추가 | enum 값 + Factory 분기 + Repository 추가 |

전략 패턴의 핵심 가치인 **"변하는 알고리즘을 캡슐화해서 중복을 제거한다"** 는 동일하다.
