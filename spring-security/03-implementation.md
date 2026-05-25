# 🔐 Spring Security 토큰 방식 — 3단계: 직접 구현하기

---

## 목차

- [1. 프로젝트 세팅](#1-프로젝트-세팅)
  - [의존성 설정](#의존성-설정)
  - [SecurityConfig 기본 구성](#securityconfig-기본-구성)
- [2. JWT 발급 구현](#2-jwt-발급-구현)
  - [JwtTokenProvider 설계](#jwttokenprovider-설계)
  - [로그인 API → Token 생성 및 응답](#로그인-api--token-생성-및-응답)
- [3. JWT 검증 필터 구현](#3-jwt-검증-필터-구현)
  - [OncePerRequestFilter 상속](#onceperrequestfilter-상속)
  - [Token 파싱 → Authentication 등록](#token-파싱--authentication-등록)
- [4. 인가(Authorization) 처리](#4-인가authorization-처리)
  - [Role 기반 접근 제어](#role-기반-접근-제어)
  - [@PreAuthorize / @Secured 활용](#preauthorize--secured-활용)

---

## 1. 프로젝트 세팅

### 의존성 설정

`build.gradle`

```groovy
dependencies {
    // Spring Security
    implementation 'org.springframework.boot:spring-boot-starter-security'

    // jjwt (JWT 라이브러리)
    implementation 'io.jsonwebtoken:jjwt-api:0.12.6'
    runtimeOnly 'io.jsonwebtoken:jjwt-impl:0.12.6'
    runtimeOnly 'io.jsonwebtoken:jjwt-jackson:0.12.6'
}
```

`application.yml`

```yaml
jwt:
  secret: "your-256-bit-secret-key-here-must-be-long-enough"  # 32자 이상 권장
  access-expiration: 1800000    # 30분 (ms)
  refresh-expiration: 604800000 # 7일 (ms)
```

---

### SecurityConfig 기본 구성

JWT 방식은 **Session을 사용하지 않고**, **CSRF 보호도 비활성화**한다.

```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthenticationFilter;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // JWT 방식 → Session 사용 안 함
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))

            // JWT는 Authorization 헤더로 전송 → CSRF 불필요
            .csrf(AbstractHttpConfigurer::disable)

            // 요청별 권한 설정
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()   // 로그인/회원가입은 인증 불필요
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )

            // JWT 필터를 UsernamePasswordAuthenticationFilter 앞에 등록
            .addFilterBefore(jwtAuthenticationFilter,
                UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
}
```

---

## 2. JWT 발급 구현

### JwtTokenProvider 설계

토큰 생성, 파싱, 검증을 전담하는 클래스다.

```java
@Component
public class JwtTokenProvider {

    private final SecretKey secretKey;
    private final long accessExpiration;
    private final long refreshExpiration;

    public JwtTokenProvider(
            @Value("${jwt.secret}") String secret,
            @Value("${jwt.access-expiration}") long accessExpiration,
            @Value("${jwt.refresh-expiration}") long refreshExpiration) {
        this.secretKey = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
        this.accessExpiration = accessExpiration;
        this.refreshExpiration = refreshExpiration;
    }

    // Access Token 생성
    public String createAccessToken(String username, String role) {
        return Jwts.builder()
            .subject(username)
            .claim("role", role)
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + accessExpiration))
            .signWith(secretKey)
            .compact();
    }

    // Refresh Token 생성
    public String createRefreshToken(String username) {
        return Jwts.builder()
            .subject(username)
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + refreshExpiration))
            .signWith(secretKey)
            .compact();
    }

    // 토큰에서 Claims 추출
    public Claims parseClaims(String token) {
        return Jwts.parser()
            .verifyWith(secretKey)
            .build()
            .parseSignedClaims(token)
            .getPayload();
    }

    // 토큰에서 username 추출
    public String getUsername(String token) {
        return parseClaims(token).getSubject();
    }

    // 토큰 유효성 검증
    public boolean validateToken(String token) {
        try {
            parseClaims(token);
            return true;
        } catch (ExpiredJwtException e) {
            throw e;  // 만료는 별도 처리 (재발급 로직)
        } catch (JwtException | IllegalArgumentException e) {
            return false;  // 위변조, 형식 오류 등
        }
    }
}
```

---

### 로그인 API → Token 생성 및 응답

**DTO**

```java
// 요청
public record LoginRequest(String email, String password) {}

// 응답
public record TokenResponse(String accessToken, String refreshToken) {}
```

**Service**

```java
@Service
@RequiredArgsConstructor
public class AuthService {

    private final AuthenticationManager authenticationManager;
    private final JwtTokenProvider jwtTokenProvider;

    public TokenResponse login(LoginRequest request) {
        // 1. 이메일/비밀번호 검증 (내부적으로 UserDetailsService 호출)
        Authentication authentication = authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(request.email(), request.password())
        );

        // 2. 인증 성공 → 토큰 생성
        String username = authentication.getName();
        String role = authentication.getAuthorities().iterator().next().getAuthority();

        String accessToken = jwtTokenProvider.createAccessToken(username, role);
        String refreshToken = jwtTokenProvider.createRefreshToken(username);

        return new TokenResponse(accessToken, refreshToken);
    }
}
```

**Controller**

```java
@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
public class AuthController {

    private final AuthService authService;

    @PostMapping("/login")
    public ResponseEntity<TokenResponse> login(@RequestBody LoginRequest request) {
        return ResponseEntity.ok(authService.login(request));
    }
}
```

**응답 예시**

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyQGV4YW1...",
  "refreshToken": "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyQGV4YW1..."
}
```

---

## 3. JWT 검증 필터 구현

### OncePerRequestFilter 상속

`OncePerRequestFilter`를 상속하면 **요청당 정확히 1번만 실행**되도록 보장된다.  
(Filter는 Forward, Redirect 등에서 중복 호출될 수 있으나 이 클래스가 이를 방지)

```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenProvider jwtTokenProvider;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
                                    throws ServletException, IOException {

        String token = extractToken(request);

        if (token != null && jwtTokenProvider.validateToken(token)) {
            // Token 파싱 → Authentication 생성 → SecurityContext 등록
            String username = jwtTokenProvider.getUsername(token);
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);

            UsernamePasswordAuthenticationToken authentication =
                new UsernamePasswordAuthenticationToken(
                    userDetails,
                    null,                          // credentials: 인증 후 null
                    userDetails.getAuthorities()
                );
            authentication.setDetails(
                new WebAuthenticationDetailsSource().buildDetails(request)
            );

            SecurityContextHolder.getContext().setAuthentication(authentication);
        }

        filterChain.doFilter(request, response);  // 다음 필터로 전달
    }

    // Authorization 헤더에서 Bearer 토큰 추출
    private String extractToken(HttpServletRequest request) {
        String bearer = request.getHeader("Authorization");
        if (StringUtils.hasText(bearer) && bearer.startsWith("Bearer ")) {
            return bearer.substring(7);
        }
        return null;
    }
}
```

### Token 파싱 → Authentication 등록

필터의 핵심 흐름을 정리하면 아래와 같다.

```
요청 수신
    ↓
Authorization 헤더에서 "Bearer <token>" 추출
    ↓
토큰 유효성 검증 (서명, 만료 확인)
    ↓  [유효]
username 파싱 → UserDetails 로드
    ↓
UsernamePasswordAuthenticationToken 생성
    ↓
SecurityContextHolder에 Authentication 등록
    ↓
다음 필터 / Controller로 진행
```

> 💡 토큰이 없거나 유효하지 않으면 `SecurityContext`에 아무것도 등록하지 않고  
> 그냥 다음 필터로 넘긴다. 이후 `AuthorizationFilter`가 인증 여부를 확인해  
> 필요 시 401을 반환한다.

---

## 4. 인가(Authorization) 처리

### Role 기반 접근 제어

**SecurityConfig에서 URL 단위로 제어**

```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/auth/**").permitAll()
    .requestMatchers(HttpMethod.GET, "/api/posts/**").permitAll()   // 조회는 모두 허용
    .requestMatchers("/api/admin/**").hasRole("ADMIN")              // ROLE_ADMIN 전용
    .requestMatchers("/api/manager/**").hasAnyRole("ADMIN", "MANAGER")
    .anyRequest().authenticated()
)
```

> 💡 `hasRole("ADMIN")`은 내부적으로 `ROLE_ADMIN`을 검사한다.  
> `UserDetails`의 `getAuthorities()`에 `ROLE_` 접두사가 포함되어 있어야 한다.

---

### @PreAuthorize / @Secured 활용

메서드 단위의 세밀한 권한 제어가 필요할 때 사용한다.  
**활성화 설정** 먼저 필요하다.

```java
@Configuration
@EnableMethodSecurity  // @PreAuthorize, @PostAuthorize, @Secured 활성화
public class SecurityConfig { ... }
```

**@PreAuthorize — 메서드 실행 전 권한 검사 (가장 많이 사용)**

```java
@RestController
@RequestMapping("/api/posts")
public class PostController {

    // ADMIN 또는 MANAGER만 접근 가능
    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN') or hasRole('MANAGER')")
    public ResponseEntity<Void> deletePost(@PathVariable Long id) { ... }

    // 본인 글만 수정 가능 (SpEL로 동적 조건)
    @PutMapping("/{id}")
    @PreAuthorize("@postService.isOwner(#id, authentication.name)")
    public ResponseEntity<Post> updatePost(@PathVariable Long id, ...) { ... }
}
```

**@PostAuthorize — 메서드 실행 후 반환값 기준으로 검사**

```java
// 반환된 데이터의 owner가 본인인 경우에만 응답
@GetMapping("/{id}")
@PostAuthorize("returnObject.owner == authentication.name")
public ResponseEntity<Post> getPost(@PathVariable Long id) { ... }
```

**@Secured — 단순 Role 체크 (SpEL 미지원)**

```java
@Secured("ROLE_ADMIN")
public void adminOnlyMethod() { ... }
```

**비교 정리**

| 애노테이션 | 검사 시점 | SpEL 지원 | 주 사용 |
|---|---|---|---|
| `@PreAuthorize` | 메서드 실행 전 | ✅ | 대부분의 경우 |
| `@PostAuthorize` | 메서드 실행 후 | ✅ | 반환값 기준 검사 |
| `@Secured` | 메서드 실행 전 | ❌ | 단순 Role 체크 |

---

## ✅ 3단계 핵심 정리

```
전체 흐름
로그인(ID/PW) → AuthService → JwtTokenProvider.createAccessToken()
                                               → 클라이언트에 토큰 반환

이후 API 요청
Authorization: Bearer <token>
    ↓
JwtAuthenticationFilter.doFilterInternal()
    └── 토큰 검증 → username 파싱 → UserDetails 로드
    └── SecurityContext에 Authentication 등록
    ↓
Controller (인증 완료 상태로 진입)

인가 처리
SecurityConfig: URL 단위 제어 (hasRole, hasAnyRole)
@PreAuthorize: 메서드 단위 세밀한 제어 (SpEL 지원)
```

---

> 📌 **이전 단계**: [2단계 — JWT 핵심 이론](./02-jwt-theory.md)  