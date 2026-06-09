# 파사드 패턴 (Facade Pattern)

GoF 디자인 패턴 중 하나로, **복잡한 서브시스템을 단순한 인터페이스 하나로 감싸는** 패턴이다.  
클라이언트는 내부 구조를 몰라도 파사드만 호출하면 된다.

---

## 구성 요소

| 역할 | 설명 |
|------|------|
| Facade | 서브시스템을 조합해 단순한 인터페이스 제공 |
| Subsystem | 실제 로직을 수행하는 개별 클래스들 |
| Client | Facade만 알고 있으며 직접 서브시스템을 호출하지 않음 |

---

## 실제 웹 개발 예시

### 시나리오

주문 요청 하나에 재고 확인 → 결제 → 배송 요청 → 알림 발송 순서로 여러 서비스가 동작한다.  
컨트롤러가 이 흐름을 직접 조합하면 의존성이 복잡해지므로, `OrderFacade`가 조합을 담당한다.

---

### 서브시스템 (Subsystem)

```java
@Service
public class InventoryService {
    public void deduct(Long productId, int quantity) {
        // 재고 차감 로직
    }
}

@Service
public class PaymentService {
    public void pay(Long orderId, int amount) {
        // 결제 처리 로직
    }
}

@Service
public class ShippingService {
    public void request(Long orderId, String address) {
        // 배송 요청 로직
    }
}

@Service
public class NotificationService {
    public void sendOrderConfirm(Long userId) {
        // 주문 완료 알림 발송
    }
}
```

---

### 파사드 (Facade)

```java
@Service
@RequiredArgsConstructor
public class OrderFacade {

    private final InventoryService inventoryService;
    private final PaymentService paymentService;
    private final ShippingService shippingService;
    private final NotificationService notificationService;

    public void placeOrder(OrderRequest request) {
        inventoryService.deduct(request.productId(), request.quantity());
        paymentService.pay(request.orderId(), request.amount());
        shippingService.request(request.orderId(), request.address());
        notificationService.sendOrderConfirm(request.userId());
    }
}
```

---

### 컨트롤러 (Client)

```java
@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderController {

    private final OrderFacade orderFacade;

    @PostMapping
    public ResponseEntity<Void> order(@RequestBody OrderRequest request) {
        orderFacade.placeOrder(request);
        return ResponseEntity.ok().build();
    }
}
```

컨트롤러는 `OrderFacade`만 알고 있으며, 내부 서비스 호출 순서를 전혀 모른다.

---

## 핵심 정리

- 파사드 패턴은 서브시스템을 숨기는 게 목적이 아니라 **복잡한 흐름을 한 곳에서 관리**하는 게 목적이다.
- 서브시스템은 여전히 독립적으로 사용 가능하다.
- 여러 서비스를 조합하는 로직이 컨트롤러나 여러 곳에 흩어져 있다면 파사드로 응집시킬 것.

---

## Facade vs UseCase

구조가 비슷해 보이지만 출신과 의도가 다르다.

| | Facade | UseCase |
|---|---|---|
| **분류** | GoF 디자인 패턴 | Clean Architecture 아키텍처 개념 |
| **목적** | 서브시스템의 복잡성 숨기기 | 비즈니스 행위 표현 |
| **내용** | 서비스 호출 조합/위임 | 비즈니스 규칙 + 유효성 검사 포함 |
| **관점** | 어떻게 감출 것인가 | 무엇을 할 것인가 |
