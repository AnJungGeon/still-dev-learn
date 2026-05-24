# 🔐 Spring Security 토큰 방식 — 1단계: 사전 지식 다지기

---

## 목차

- [1. Spring Security 기본 구조 이해](#1-spring-security-기본-구조-이해)
  - [FilterChain 동작 원리](#filterchain-동작-원리)
  - [SecurityContext / Authentication 객체](#securitycontext--authentication-객체)
  - [UserDetails / UserDetailsService](#userdetails--userdetailsservice)
- [2. HTTP 인증 방식 비교](#2-http-인증-방식-비교)
  - [Session 방식 vs Token 방식](#session-방식-vs-token-방식)
  - [ThreadLocal 전략](#threadlocal-전략)

---

## 1. Spring Security 기본 구조 이해

### FilterChain 동작 원리

Spring Security는 **Servlet Filter** 기반으로 동작한다.  
HTTP 요청이 들어오면 DispatcherServlet에 도달하기 전에 **FilterChain**을 먼저 통과한다.

```
HTTP 요청
    ↓
[DelegatingFilterProxy]         ← Servlet Container가 인식
    ↓
[FilterChainProxy]              ← Spring이 관리하는 Security Filter들의 진입점
    ↓
[SecurityFilterChain]
    ├── DisableEncodeUrlFilter
    ├── SecurityContextHolderFilter
    ├── UsernamePasswordAuthenticationFilter   ← 폼 로그인 처리
    ├── ExceptionTranslationFilter             ← 인증/인가 예외 변환
    ├── AuthorizationFilter                    ← 접근 권한 검사
    └── ...
    ↓
DispatcherServlet → Controller
```

**핵심 구성 요소**

| 구성 요소 | 역할 |
|---|---|
| `DelegatingFilterProxy` | Servlet Container와 Spring ApplicationContext를 연결하는 다리 |
| `FilterChainProxy` | Spring Security의 실제 진입점. 여러 `SecurityFilterChain`을 관리 |
| `SecurityFilterChain` | 특정 URL 패턴에 적용할 필터 목록을 묶은 단위 |

> 💡 Spring Security 6 기준 `WebSecurityConfigurerAdapter`는 deprecated.  
> `SecurityFilterChain`을 **Bean으로 직접 등록**하는 방식을 사용한다.

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/public/**").permitAll()
            .anyRequest().authenticated()
        );
    return http.build();
}
```

---

### SecurityContext / Authentication 객체

인증이 완료되면 인증 정보는 **SecurityContextHolder** 안의 **SecurityContext**에 저장된다.

```
SecurityContextHolder
    └── SecurityContext
            └── Authentication
                    ├── Principal        (현재 사용자 — UserDetails 객체)
                    ├── Credentials      (비밀번호, 토큰 등 — 인증 후 보통 null로 초기화)
                    ├── Authorities      (권한 목록 — ROLE_USER, ROLE_ADMIN 등)
                    └── isAuthenticated  (인증 여부)
```

**SecurityContextHolder 전략 (기본: ThreadLocal)**

```java
// 어디서든 현재 인증 정보 꺼내기
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
String username = auth.getName();
Collection<? extends GrantedAuthority> roles = auth.getAuthorities();
```

> 💡 ThreadLocal 전략이므로 동일 스레드 안에서는 어디서든 인증 정보에 접근 가능.  
> 요청이 끝나면 컨텍스트를 **clear** 해야 메모리 누수를 막을 수 있다.  
> (Spring Security가 `SecurityContextHolderFilter`에서 자동으로 처리)

**Authentication 주요 구현체**

| 구현체 | 사용 시점 |
|---|---|
| `UsernamePasswordAuthenticationToken` | 폼 로그인 / 일반 인증 |
| `AnonymousAuthenticationToken` | 인증되지 않은 요청 |
| `JwtAuthenticationToken` | JWT 기반 인증 (직접 구현하거나 라이브러리 사용) |

---

### UserDetails / UserDetailsService

Spring Security가 사용자 정보를 로드할 때 사용하는 **표준 인터페이스**다.

**UserDetails — 사용자 정보 표현**

```java
public interface UserDetails extends Serializable {
    Collection<? extends GrantedAuthority> getAuthorities(); // 권한 목록
    String getPassword();                                    // 인코딩된 비밀번호
    String getUsername();                                    // 사용자 식별자
    boolean isAccountNonExpired();                           // 계정 만료 여부
    boolean isAccountNonLocked();                            // 계정 잠금 여부
    boolean isCredentialsNonExpired();                       // 자격증명 만료 여부
    boolean isEnabled();                                     // 활성화 여부
}
```

**UserDetailsService — DB에서 사용자 로드**

```java
public interface UserDetailsService {
    UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

**실제 구현 예시**

```java
@Service
@RequiredArgsConstructor
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByEmail(username)
            .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));

        return org.springframework.security.core.userdetails.User.builder()
            .username(user.getEmail())
            .password(user.getPassword())   // BCrypt 인코딩된 값
            .roles(user.getRole().name())   // ROLE_ 접두사 자동 추가
            .build();
    }
}
```

> 💡 JWT 방식에서는 매 요청마다 `loadUserByUsername`을 호출해 DB를 조회하는 대신,  
> 토큰 안의 Claim에서 사용자 정보를 꺼내 Authentication을 직접 생성하는 패턴도 많이 쓴다.

---

## 2. HTTP 인증 방식 비교

### Session 방식 vs Token 방식

**Session 방식 (Stateful)**

```
1. 클라이언트 → 서버: 로그인 (ID/PW)
2. 서버 → 세션 저장소: 세션 생성 (세션 ID 발급)
3. 서버 → 클라이언트: Set-Cookie: JSESSIONID=xxxx
4. 클라이언트 → 서버: 이후 요청마다 Cookie 자동 전송
5. 서버 → 세션 저장소: 세션 ID로 사용자 조회 → 인증 처리
```

**Token 방식 (Stateless)**

```
1. 클라이언트 → 서버: 로그인 (ID/PW)
2. 서버: 토큰(JWT) 생성
3. 서버 → 클라이언트: Access Token (+ Refresh Token)
4. 클라이언트 → 서버: 이후 요청마다 Authorization: Bearer <token>
5. 서버: 토큰 서명 검증 → 인증 처리 (세션 저장소 조회 없음)
```

**비교표**

| 항목 | Session 방식 | Token(JWT) 방식 |
|---|---|---|
| **상태 관리** | Stateful (서버가 세션 저장) | Stateless (서버가 상태 없음) |
| **서버 확장성** | 낮음 (세션 공유 문제) | 높음 (어떤 서버든 검증 가능) |
| **저장 위치** | 서버 메모리 / Redis | 클라이언트 (LocalStorage / Cookie) |
| **전송 방식** | Cookie 자동 전송 | Authorization 헤더 수동 전송 |
| **무효화** | 서버에서 즉시 삭제 가능 | 만료 전 강제 무효화 어려움 |
| **보안 위협** | CSRF 취약 | XSS 취약 (저장 위치에 따라) |
| **주 사용처** | 전통적인 웹 서비스 | REST API, MSA, 모바일 앱 |

---

### ThreadLocal 전략

#### 쓰레드 1개 = 요청 1개 = 사용자 1개 권한 관리

Spring MVC(Tomcat 기준)는 기본적으로 **Thread-per-Request** 모델로 동작한다.  
HTTP 요청이 들어오면 스레드 풀에서 스레드 하나를 꺼내 해당 요청을 전담 처리하고, 응답이 끝나면 스레드를 반납한다.

```
[요청 A — 사용자 Kim]  →  Thread-1  →  SecurityContext { Authentication: Kim }
[요청 B — 사용자 Lee]  →  Thread-2  →  SecurityContext { Authentication: Lee }
[요청 C — 사용자 Park] →  Thread-3  →  SecurityContext { Authentication: Park }
```

`SecurityContextHolder`는 기본 전략으로 **`ThreadLocal`** 을 사용한다.  
`ThreadLocal`은 같은 스레드 안에서만 값을 공유하는 저장소이므로, 요청마다 인증 정보가 완벽히 격리된다.

```java
// 내부적으로 ThreadLocal에 저장
SecurityContextHolder.getContext().setAuthentication(authentication);

// 동일 스레드(= 동일 요청) 어디서든 꺼낼 수 있다
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
```

> ⚠️ 요청이 끝난 뒤 `SecurityContextHolder.clearContext()`를 호출하지 않으면  
> 스레드가 재사용될 때 이전 사용자의 인증 정보가 남아 있을 수 있다.  
> Spring Security의 `SecurityContextHolderFilter`가 요청 완료 후 자동으로 clear한다.

---

## ✅ 1단계 핵심 정리

```
Spring Security 요청 흐름
HTTP 요청 → FilterChainProxy → SecurityFilterChain → (인증/인가) → Controller

인증 정보 저장
SecurityContextHolder(ThreadLocal) → SecurityContext → Authentication → Principal(UserDetails)

ThreadLocal 격리 원칙
쓰레드 1개 = 요청 1개 = 사용자 1개 → 인증 정보가 요청 간 절대 섞이지 않음
요청 완료 후 SecurityContextHolderFilter가 자동으로 clear

사용자 로드
UserDetailsService.loadUserByUsername() → UserDetails 반환

인증 방식 결정 기준
- 전통적 웹(SSR): Session 방식
- REST API / SPA / 모바일: Token(JWT) 방식
```

---

> 📌 **다음 단계**: [2단계 — JWT 핵심 이론](./02-jwt-theory.md)
