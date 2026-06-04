# TypeScript — Nullish / Nullable / Option 패턴

---

## 목차

- [Nullable 타입](#nullable-타입)
- [Nullish 연산자](#nullish-연산자)
- [Option 패턴](#option-패턴)

---

## Nullable 타입

### strictNullChecks

`tsconfig.json`에서 `strictNullChecks: true`(기본값)이면  
`null`과 `undefined`는 각 타입에 자동으로 포함되지 않는다.

```typescript
// strictNullChecks: true (기본)
let name: string = null;   // ❌ 컴파일 오류
let name: string = undefined; // ❌ 컴파일 오류

// null을 허용하려면 명시해야 한다
let name: string | null = null;       // ✅
let name: string | undefined = undefined; // ✅
let name: string | null | undefined = null; // ✅
```

### 함수 파라미터에서

```typescript
// null/undefined를 받을 수 없음
function greet(name: string) {
    return `Hello, ${name}`;
}

greet(null); // ❌

// 허용하려면 명시
function greet(name: string | null) {
    if (name === null) return "Hello, stranger";
    return `Hello, ${name}`;
}
```

### 타입 가드로 좁히기

```typescript
function printLength(value: string | null | undefined) {
    if (value == null) {  // null과 undefined 동시에 체크 (느슨한 비교)
        console.log("값 없음");
        return;
    }
    console.log(value.length); // 여기서 value는 string으로 좁혀짐
}
```

---

## Nullish 연산자

### `?.` 옵셔널 체이닝

프로퍼티/메서드 접근 시 중간에 `null | undefined`가 있으면 **오류 없이 undefined를 반환**한다.

```typescript
const user = {
    profile: {
        address: {
            city: "Seoul"
        }
    }
};

// 기존 방식
const city = user && user.profile && user.profile.address && user.profile.address.city;

// 옵셔널 체이닝
const city = user?.profile?.address?.city; // "Seoul"
const zip  = user?.profile?.address?.zip;  // undefined (오류 없음)
```

메서드 호출에도 사용 가능하다.

```typescript
const result = someArray?.find(x => x.id === 1);
const upper  = str?.toUpperCase();
```

### `??` Nullish 병합 연산자

좌측이 `null | undefined`일 때만 우측 값을 사용한다.  
`||`와의 차이: `||`는 falsy(`0`, `""`, `false`)도 우측으로 넘어가지만, `??`는 오직 `null | undefined`만 해당한다.

```typescript
const count = null ?? 0;      // 0
const count = undefined ?? 0; // 0
const count = 0 ?? 42;        // 0  ← || 였다면 42가 됨
const count = "" ?? "기본값"; // "" ← || 였다면 "기본값"이 됨
```

```typescript
// 실전 예시
function getDisplayName(name: string | null | undefined): string {
    return name ?? "익명";
}
```

### `??=` Nullish 할당 연산자

좌측이 `null | undefined`일 때만 우측 값을 **할당**한다.

```typescript
let cache: string | null = null;
cache ??= "computed value"; // null이므로 할당됨
cache ??= "other value";    // 이미 값이 있으므로 무시됨

console.log(cache); // "computed value"
```

### `?.` + `??` 조합

```typescript
const city = user?.profile?.address?.city ?? "도시 정보 없음";
```

---

## Option 패턴

TypeScript에는 Kotlin의 `?.let {}`처럼 **값이 있을 때만 연산을 체이닝**하는 내장 문법이 없다.  
이를 직접 구현하면 null 처리 로직을 한 곳으로 모을 수 있다.

### 직접 구현

```typescript
class Option<T> {
    private constructor(private readonly value: T | null | undefined) {}

    static of<T>(value: T | null | undefined): Option<T> {
        return new Option(value);
    }

    // 값이 있을 때만 변환
    map<U>(fn: (value: T) => U): Option<U> {
        if (this.value == null) return Option.of<U>(null);
        return Option.of(fn(this.value));
    }

    // 값이 있을 때만 실행 (부수효과)
    ifPresent(fn: (value: T) => void): this {
        if (this.value != null) fn(this.value);
        return this;
    }

    // 값이 없으면 기본값 반환
    getOrElse(defaultValue: T): T {
        return this.value ?? defaultValue;
    }

    // 값이 있는지 여부
    isPresent(): boolean {
        return this.value != null;
    }

    // 값 꺼내기 (없으면 오류)
    get(): T {
        if (this.value == null) throw new Error("Option is empty");
        return this.value;
    }
}
```

### 사용 예시

```typescript
const userName: string | null = getUserFromDB(); // null일 수도 있음

// 기존 방식 — if 체크 반복
if (userName !== null) {
    const upper = userName.toUpperCase();
    if (upper.length > 0) {
        console.log(upper);
    }
}

// Option 패턴
Option.of(getUserFromDB())
    .map(name => name.toUpperCase())
    .ifPresent(name => console.log(name));

// 기본값 처리
const displayName = Option.of(getUserFromDB())
    .map(name => name.trim())
    .getOrElse("익명 사용자");
```

### `?.` + Option 패턴 비교

```typescript
// 옵셔널 체이닝 — 단순 접근에 적합
const city = user?.profile?.address?.city ?? "없음";

// Option 패턴 — 변환/가공 로직이 많을 때 적합
const displayCity = Option.of(user)
    .map(u => u.profile)
    .map(p => p.address)
    .map(a => a.city.toUpperCase())
    .getOrElse("도시 정보 없음");
```

---

## 핵심 정리

```
Nullable
string | null       → null만 허용
string | undefined  → undefined만 허용
string | null | undefined → 둘 다 허용

연산자
?.   값이 null/undefined면 접근 중단 → undefined 반환
??   좌측이 null/undefined면 우측 사용 (0, ""는 그대로)
??=  좌측이 null/undefined일 때만 우측을 할당

Option 패턴
null 체크 로직을 캡슐화해 map/getOrElse로 체이닝
변환 단계가 많을수록 가독성 이점이 크다
```
