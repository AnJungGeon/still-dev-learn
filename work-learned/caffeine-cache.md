# Caffeine Cache

## 개요

Caffeine은 Java 8 기반의 고성능 in-memory 캐시 라이브러리.

- Spring Boot 2.x 이상에서 기본 로컬 캐시로 사용

---

## 의존성

```gradle
// Gradle
implementation 'com.github.ben-manes.caffeine:caffeine:3.1.8'

// Spring Boot Cache와 함께 사용 시
implementation 'org.springframework.boot:spring-boot-starter-cache'
```




---

## 기본 사용법

```java
Cache<String, String> cache = Caffeine.newBuilder()
    .maximumSize(1000)
    .expireAfterWrite(10, TimeUnit.MINUTES)
    .build();

// 저장
cache.put("key", "value");

// 조회 (없으면 null)
String value = cache.getIfPresent("key");

// 조회 + 없으면 로드
String value = cache.get("key", k -> loadFromDb(k));

// 삭제
cache.invalidate("key");
cache.invalidateAll();
```

---

## 주요 설정 옵션

### 크기 기반 만료

```java
Caffeine.newBuilder()
    .maximumSize(10_000)          // 최대 항목 수
    .maximumWeight(1_000_000)     // 최대 가중치 (weigher와 함께 사용)
    .weigher((key, value) -> value.size())
```

### 시간 기반 만료

```java
Caffeine.newBuilder()
    .expireAfterWrite(10, TimeUnit.MINUTES)   // 쓴 후 N분 뒤 만료
    .expireAfterAccess(5, TimeUnit.MINUTES)   // 마지막 접근 후 N분 뒤 만료
    .refreshAfterWrite(1, TimeUnit.MINUTES)   // 쓴 후 N분 뒤 갱신 (백그라운드)
```

| 옵션 | 설명 |
|------|------|
| `expireAfterWrite` | 캐시에 쓴 시점 기준으로 만료 |
| `expireAfterAccess` | 마지막 읽기/쓰기 기준으로 만료 (자주 쓰는 항목은 살아남음) |
| `refreshAfterWrite` | 만료되지 않고 백그라운드에서 갱신 (stale 데이터 허용) |

### 참조 기반 만료

```java
Caffeine.newBuilder()
    .weakKeys()     // GC 수집 대상 키
    .weakValues()   // GC 수집 대상 값
    .softValues()   // 메모리 부족 시 GC 수집
```

---

## 로딩 캐시 (LoadingCache)

캐시 미스 시 자동으로 값을 로드하는 캐시.

```java
LoadingCache<String, User> cache = Caffeine.newBuilder()
    .maximumSize(1000)
    .expireAfterWrite(10, TimeUnit.MINUTES)
    .build(key -> userRepository.findById(key));  // 캐시 미스 시 자동 호출

// 사용 - 없으면 자동 로드
User user = cache.get("userId");

// 복수 키 조회
Map<String, User> users = cache.getAll(List.of("id1", "id2"));
```

---

## 비동기 캐시 (AsyncCache)

캐시 미스 시 비동기로 값을 로드하며, `CompletableFuture`를 반환하는 캐시.

```java
AsyncLoadingCache<String, User> asyncCache = Caffeine.newBuilder()
    .maximumSize(1000)
    .buildAsync(key -> fetchUserAsync(key));  // CompletableFuture 반환

CompletableFuture<User> future = asyncCache.get("userId");
```

---

## Spring Boot 연동

### 설정 클래스

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager();
        manager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(500)
            .expireAfterWrite(10, TimeUnit.MINUTES)
            .recordStats());
        return manager;
    }
}
```

### application.yml 방식

```yaml
spring:
  cache:
    type: caffeine
    caffeine:
      spec: maximumSize=500,expireAfterWrite=10m
    cache-names:
      - users
      - products
```

### 어노테이션 사용

```java
@Service
public class UserService {

    @Cacheable(value = "users", key = "#userId")
    public User getUser(String userId) {
        return userRepository.findById(userId);
    }

    @CachePut(value = "users", key = "#user.id")
    public User updateUser(User user) {
        return userRepository.save(user);
    }

    @CacheEvict(value = "users", key = "#userId")
    public void deleteUser(String userId) {
        userRepository.deleteById(userId);
    }

    @CacheEvict(value = "users", allEntries = true)
    public void clearAllUsers() {}
}
```

---

## 캐시별 다른 설정 적용

```java
@Bean
public CacheManager cacheManager() {
    SimpleCacheManager manager = new SimpleCacheManager();
    
    List<CaffeineCache> caches = List.of(
        buildCache("users", 1000, 10),
        buildCache("products", 5000, 30),
        buildCache("sessions", 500, 1)
    );
    
    manager.setCaches(caches);
    return manager;
}

private CaffeineCache buildCache(String name, int maxSize, int minutesToExpire) {
    return new CaffeineCache(name,
        Caffeine.newBuilder()
            .maximumSize(maxSize)
            .expireAfterWrite(minutesToExpire, TimeUnit.MINUTES)
            .build());
}
```

---

## 통계 모니터링

```java
Cache<String, String> cache = Caffeine.newBuilder()
    .maximumSize(1000)
    .recordStats()   // 통계 수집 활성화
    .build();

CacheStats stats = cache.stats();
System.out.println("히트율: " + stats.hitRate());
System.out.println("히트 수: " + stats.hitCount());
System.out.println("미스 수: " + stats.missCount());
System.out.println("로드 횟수: " + stats.loadCount());
System.out.println("평균 로드 시간: " + stats.averageLoadPenalty());
System.out.println("만료 수: " + stats.evictionCount());
```

---

## 제거 리스너 (Eviction Listener)

```java
Cache<String, String> cache = Caffeine.newBuilder()
    .maximumSize(1000)
    .evictionListener((key, value, cause) -> {
        log.info("캐시 제거: key={}, cause={}", key, cause);
        // cause: SIZE, EXPIRED, EXPLICIT, REPLACED, COLLECTED
    })
    .removalListener((key, value, cause) -> {
        // eviction + 명시적 삭제 모두 감지
    })
    .build();
```

---

## 주의사항

- `expireAfterWrite` vs `expireAfterAccess`: 정합성이 중요하면 `Write`, 메모리 효율이 중요하면 `Access`
- `refreshAfterWrite`는 만료가 아니므로 stale 데이터 반환 가능 — 실시간 정합성 필요 시 부적합
- `weakValues` / `softValues` 사용 시 GC 타이밍에 의존하므로 동작 예측이 어려움
- Spring `@Cacheable`의 `key`가 없으면 메서드 파라미터 전체가 키 — 복잡한 객체는 명시적으로 지정
