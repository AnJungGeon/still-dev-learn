# Zod 정리

Zod는 **런타임 유효성 검사** 라이브러리다.  
스키마를 정의하면 런타임에 실제로 값을 검증하고, TypeScript 타입도 자동으로 추론해준다.

---

## 목차

- [nullable / optional / nullish](#nullable--optional--nullish)

---

## nullable / optional / nullish

순수 TypeScript의 타입 표기와 달리, Zod는 **메서드 체이닝**으로 null/undefined 허용 여부를 제어한다.

| 메서드 | 허용 타입 | 동등한 표현 |
|---|---|---|
| `.nullable()` | `T \| null` | — |
| `.optional()` | `T \| undefined` | — |
| `.nullish()` | `T \| null \| undefined` | `.nullable().optional()` |

```typescript
import { z } from "zod";

const a = z.string().nullable();  // string | null
const b = z.string().optional();  // string | undefined
const c = z.string().nullish();   // string | null | undefined

// .nullish()는 아래와 완전히 동일하다
const c2 = z.string().nullable().optional();
```

순서를 바꿔도 결과는 같다.

```typescript
z.string().nullable().optional(); // string | null | undefined
z.string().optional().nullable(); // string | null | undefined  ← 동일
```

**TypeScript 타입과의 차이**

```typescript
// TypeScript — 컴파일 타임 타입 선언만 (런타임엔 흔적 없음)
let name: string | null;

// Zod — 런타임에 실제로 null을 허용하는 검증 로직 + 타입 추론
const nameSchema = z.string().nullable();
type Name = z.infer<typeof nameSchema>; // string | null
```

> 💡 `.nullish()`는 `null`과 `undefined` 둘 다 허용해야 할 때 쓰는 편의 메서드.  
> 내부적으로 `.nullable().optional()`과 동일하게 동작한다.
