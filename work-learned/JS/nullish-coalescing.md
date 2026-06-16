# Nullish Coalescing (`??`)

## falsy vs nullish

`||`는 `0`, `''` 같은 값도 falsy로 보고 기본값으로 덮어쓰지만, `??`는 `null` / `undefined` 일 때만 기본값으로 교체한다.

```js
// falsy: false, 0, '', null, undefined, NaN
0 || '기본값'        // '기본값'  ← 0도 falsy라 교체됨
0 ?? '기본값'        // 0        ← null/undefined가 아니므로 그대로

'' || '이름 없음'    // '이름 없음'
'' ?? '이름 없음'    // ''

null || '기본값'     // '기본값'
null ?? '기본값'     // '기본값'

undefined || '기본값'  // '기본값'
undefined ?? '기본값'  // '기본값'
```

### 언제 뭘 쓸까

API 응답 데이터를 화면에 표시할 때 기준으로 생각하면 편하다.

```js
// 서버에서 받아온 데이터
const product = { name: '', price: 0, stock: null }

// 이름 — 빈 문자열도 서버가 보낸 값이므로 그대로 표시
product.name ?? '상품명 없음'   // '' → '' (서버 값 유지)
product.name || '상품명 없음'   // '' → '상품명 없음' (의도와 다름)

// 가격 — 0원도 유효한 가격
product.price ?? '가격 미정'    // 0 → 0
product.price || '가격 미정'    // 0 → '가격 미정' (의도와 다름)

// 재고 — null이면 진짜 데이터 없음
product.stock ?? '재고 미확인'  // null → '재고 미확인'
```


---

## `?.` 옵셔널 체이닝과 함께

```js
const city = user?.address?.city ?? '미입력'
// user나 address가 null/undefined면 '미입력'
```

---

## `??=` 널 병합 할당

값이 `null` / `undefined` 일 때만 할당한다.

```js
let name = null;
name ??= '익명';   // name = '익명'

let count = 0;
count ??= 10;      // count = 0  (0은 nullish가 아님)
```
