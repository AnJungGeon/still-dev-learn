# putIfAbsent

`Map`에 키가 없을 때만 값을 넣는다. 있으면 아무것도 안 한다.

```java
map.putIfAbsent(key, value);
```

---

## put과의 차이

```java
Map<String, String> map = new HashMap<>();
map.put("a", "기존값");

map.put("a", "새값");           // 덮어씀 → "새값"
map.putIfAbsent("a", "새값");   // 무시됨 → 여전히 "기존값"
```

---

## 반환값

- 키가 **이미 있으면** 기존 값 반환
- 키가 **없었으면** null 반환 (넣고 나서)

```java
String prev = map.putIfAbsent("b", "첫값");
// prev == null (없었으므로)

String prev2 = map.putIfAbsent("b", "두번째값");
// prev2 == "첫값" (이미 있었으므로)
```

---

## 주요 사용 패턴

### 기본값 보장

```java
// 키가 없을 때만 초기값 세팅
map.putIfAbsent("count", 0);
```

### 중복 등록 방지

```java
// 이미 등록된 핸들러면 무시
handlers.putIfAbsent(eventType, defaultHandler);
```

### ConcurrentHashMap에서 스레드 안전 초기화

```java
ConcurrentHashMap<String, List<String>> map = new ConcurrentHashMap<>();

// 멀티스레드 환경에서 안전하게 리스트 초기화
map.putIfAbsent("key", new ArrayList<>());
map.get("key").add("item");
```

> `ConcurrentHashMap.putIfAbsent`는 원자적(atomic) 연산이라 동기화 블록 없이도 안전하다.  
> 단, `putIfAbsent` 후 `get` + `add`는 원자적이지 않으므로 복합 연산은 `computeIfAbsent`를 쓴다.

---

## computeIfAbsent와 비교

```java
// putIfAbsent: 값을 미리 만들어서 넘김 (키가 있어도 객체 생성됨)
map.putIfAbsent("key", new ArrayList<>());

// computeIfAbsent: 키가 없을 때만 람다 실행 (객체 생성 비용 절약)
map.computeIfAbsent("key", k -> new ArrayList<>());
```

값 생성 비용이 있거나 사이드 이펙트가 있는 경우 `computeIfAbsent`가 낫다.
