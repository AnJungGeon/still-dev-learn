# Boolean 축약 표현 정리

## 1. 값 반전 (toggle)

```ts
// 길게
isLoggedIn === true ? false : true

// 짧게
!isLoggedIn
```

## 2. boolean 변환 (truthy → true)

```ts
// 길게
username ? true : false

// 짧게
!!username
// undefined / null / 0 / '' → false
// 나머지 → true
```

## 3. 기본값 처리

```ts
// 길게
userName !== null && userName !== undefined ? userName : '이름 없음'

// 짧게
userName ?? '이름 없음'
```

## 4. 조건부 실행

```ts
// 길게
if (isLoggedIn === true) {
  openDashboard();
}

// 짧게
isLoggedIn && openDashboard();
```

## 5. 조건부 값 할당

```ts
// 길게
if (isAdmin) {
  label = '관리자';
} else {
  label = '일반 유저';
}

// 짧게
label = isAdmin ? '관리자' : '일반 유저';
```