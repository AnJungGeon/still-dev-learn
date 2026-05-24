# 🔐 Spring Security 토큰 방식 — 2단계: JWT 핵심 이론

---

## 목차

- [1. JWT 구조 분석](#1-jwt-구조-분석)
  - [Header / Payload / Signature](#header--payload--signature)
  - [Claims 종류](#claims-종류)
- [2. JWT 서명 알고리즘](#2-jwt-서명-알고리즘)
  - [HS256 (대칭키) vs RS256 (비대칭키)](#hs256-대칭키-vs-rs256-비대칭키)
- [3. Access Token / Refresh Token 전략](#3-access-token--refresh-token-전략)
  - [이중 토큰 구조의 필요성](#이중-토큰-구조의-필요성)
  - [Token 만료 및 재발급 흐름](#token-만료-및-재발급-흐름)

---

## 1. JWT 구조 분석

### Header / Payload / Signature

JWT는 `.`으로 구분된 **3개의 Base64URL 인코딩 문자열**로 이루어진다.

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9   ← Header
.
eyJzdWIiOiJ1c2VyMTIzIiwicm9sZSI6IlVTRVIiLCJleHAiOjE3MDAwMDAwMDB9  ← Payload
.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c  ← Signature
```

> ⚠️ Base64URL은 **인코딩**이지 **암호화가 아니다**.  
> Header와 Payload는 누구나 디코딩해서 내용을 볼 수 있으므로  
> **비밀번호, 개인정보 등 민감한 데이터는 절대 넣지 않는다.**

---

**Header**

토큰의 타입과 서명 알고리즘을 명시한다.

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

---

**Payload**

실제 전달할 데이터(Claims)를 담는다.

```json
{
  "sub": "user123",
  "role": "USER",
  "iat": 1700000000,
  "exp": 1700003600
}
```

---

**Signature**

Header와 Payload가 위조되지 않았음을 검증하는 서명이다.

```
Signature = HMACSHA256(
  Base64URL(Header) + "." + Base64URL(Payload),
  secretKey
)
```

서버는 수신한 토큰을 동일한 알고리즘과 키로 다시 서명해 비교함으로써 **위변조 여부를 검증**한다.

---

### Claims 종류

JWT Payload에 담기는 데이터 항목을 **Claim**이라 한다.  
크게 **등록된 Claim(Registered)**, **공개 Claim(Public)**, **비공개 Claim(Private)**으로 구분한다.

**등록된 Claim (Registered Claims)**

표준으로 정의된 Claim. 사용은 선택이지만 의미가 약속되어 있다.

| Claim | 의미 | 예시 |
|---|---|---|
| `iss` | Issuer — 토큰 발급자 | `"iss": "my-service"` |
| `sub` | Subject — 토큰 주체 (보통 사용자 ID) | `"sub": "user123"` |
| `aud` | Audience — 토큰 수신 대상 | `"aud": "my-client"` |
| `exp` | Expiration — 만료 시각 (Unix timestamp) | `"exp": 1700003600` |
| `nbf` | Not Before — 이 시각 이전에는 유효하지 않음 | `"nbf": 1700000000` |
| `iat` | Issued At — 발급 시각 | `"iat": 1700000000` |
| `jti` | JWT ID — 토큰 고유 식별자 (중복 방지) | `"jti": "abc-123"` |

**비공개 Claim (Private Claims)**

서버와 클라이언트 간 약속한 커스텀 데이터. 서비스 요구사항에 맞게 자유롭게 추가한다.

```json
{
  "sub": "user123",
  "role": "ADMIN",
  "email": "user@example.com",
  "exp": 1700003600
}
```

---

## 2. JWT 서명 알고리즘

### HS256 (대칭키) vs RS256 (비대칭키)

**HS256 — HMAC + SHA-256 (대칭키)**

발급(서명)과 검증에 **같은 비밀키**를 사용한다.

```
[서버] 비밀키로 서명 → JWT 발급
[서버] 동일한 비밀키로 서명 검증
```

```java
// 서명
String token = Jwts.builder()
    .subject("user123")
    .signWith(Keys.hmacShaKeyFor(secretKeyBytes))
    .compact();

// 검증
Claims claims = Jwts.parser()
    .verifyWith(Keys.hmacShaKeyFor(secretKeyBytes))
    .build()
    .parseSignedClaims(token)
    .getPayload();
```

| 항목 | 내용 |
|---|---|
| **키 구조** | 단일 비밀키 (Secret Key) |
| **속도** | 빠름 |
| **적합한 환경** | 단일 서버, 내부 서비스 간 통신 |
| **단점** | 비밀키를 공유한 모든 서버가 토큰을 발급할 수 있음 |

---

**RS256 — RSA + SHA-256 (비대칭키)**

**개인키(Private Key)**로 서명하고, **공개키(Public Key)**로 검증한다.

```
[인증 서버] 개인키로 서명 → JWT 발급
[리소스 서버] 공개키로 서명 검증 (발급 불가)
```

| 항목 | 내용 |
|---|---|
| **키 구조** | 개인키(서명) + 공개키(검증) |
| **속도** | HS256보다 느림 |
| **적합한 환경** | MSA, OAuth 2.0, 외부 공개 API |
| **장점** | 공개키만 배포하면 되므로 검증 서버가 토큰을 발급할 수 없음 |

---

**선택 기준 요약**

```
단일 서버 / 사내 서비스          → HS256 (단순하고 빠름)
MSA / 인증 서버 분리 / OAuth 2.0 → RS256 (발급과 검증 역할 분리)
```

---

## 3. Access Token / Refresh Token 전략

### 이중 토큰 구조의 필요성

JWT는 한 번 발급되면 **만료 전 강제 무효화가 불가**하다.  
토큰이 탈취되더라도 서버는 이를 즉시 막을 방법이 없다.

이 문제를 완화하기 위해 **수명이 다른 두 토큰**을 함께 사용한다.

| 토큰 | 역할 | 만료 시간 | 저장 위치 |
|---|---|---|---|
| **Access Token** | API 요청 시 인증에 사용 | 짧음 (15분 ~ 1시간) | 메모리 또는 쿠키 |
| **Refresh Token** | Access Token 재발급 요청에 사용 | 길음 (7일 ~ 30일) | HttpOnly Cookie 또는 DB/Redis |

- Access Token 수명이 짧으므로 탈취되더라도 **피해 시간이 제한**된다.
- Refresh Token은 API 요청에 직접 노출되지 않으므로 **탈취 가능성이 낮다.**
- Refresh Token은 DB/Redis에 저장해두면 **서버 측 강제 만료**가 가능하다.

---

### Token 만료 및 재발급 흐름

**정상 흐름**

```
1. 클라이언트 → 서버: 로그인 (ID/PW)
2. 서버 → 클라이언트: Access Token (짧은 만료) + Refresh Token (긴 만료)

3. 클라이언트 → 서버: API 요청 (Authorization: Bearer <AccessToken>)
4. 서버: Access Token 검증 → 200 OK 응답
```

**Access Token 만료 시 재발급 흐름**

```
1. 클라이언트 → 서버: API 요청 (만료된 Access Token)
2. 서버 → 클라이언트: 401 Unauthorized

3. 클라이언트 → 서버: 재발급 요청 (Refresh Token 전송)
4. 서버: Refresh Token 검증 (유효성 + DB 저장값 비교)
5. 서버 → 클라이언트: 새로운 Access Token 발급

6. 클라이언트 → 서버: 원래 API 재요청 (새 Access Token)
7. 서버: 200 OK 응답
```

**Refresh Token도 만료된 경우**

```
3. 클라이언트 → 서버: 재발급 요청 (만료된 Refresh Token)
4. 서버 → 클라이언트: 401 Unauthorized
5. 클라이언트: 로그인 페이지로 이동 (재로그인 필요)
```

---

**RTR (Refresh Token Rotation) — 보안 강화**

Refresh Token을 **사용할 때마다 새것으로 교체**하는 전략.  
탈취된 토큰이 사용되면 기존 토큰이 이미 무효화되어 있으므로 탐지가 가능하다.

```
1. 클라이언트 → 서버: Access Token 재발급 요청 (Refresh Token A 전송)
2. 서버: Refresh Token A 검증 → 무효화
3. 서버 → 클라이언트: 새 Access Token + 새 Refresh Token B 발급

4. (만약 공격자가 탈취한 Refresh Token A로 요청 시)
   서버: A는 이미 무효화 → 401 Unauthorized (탈취 탐지)
```

---

## ✅ 2단계 핵심 정리

```
JWT 구조
Header (alg, typ) + Payload (Claims) + Signature
→ Base64URL 인코딩 — 암호화 아님, 민감 정보 절대 포함 금지

주요 Claims
sub(사용자 ID), exp(만료), iat(발급 시각), jti(고유 ID)

서명 알고리즘
HS256: 단일 비밀키, 단순/빠름, 단일 서버에 적합
RS256: 개인키(서명) + 공개키(검증), MSA/OAuth에 적합

이중 토큰 전략
Access Token: 짧은 수명, API 인증에 사용
Refresh Token: 긴 수명, Access Token 재발급에만 사용
RTR: Refresh Token 사용 시마다 교체 → 탈취 탐지 가능
```

---

> 📌 **이전 단계**: [1단계 — 사전 지식 다지기](./01-prerequisites.md)  
> 📌 **다음 단계**: [3단계 — 직접 구현하기](./03-implementation.md)
