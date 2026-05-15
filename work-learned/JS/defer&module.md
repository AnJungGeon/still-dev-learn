# `<script defer>` 속성과 JS 모듈

## 1. defer 속성 기본 개념

### 정의
`defer` 속성은 스크립트 파일을 **비동기로 다운로드**하되, **HTML 파싱이 완료된 후 실행**되도록 하는 속성입니다.

```html
<script src="app.js" defer></script>
```

### 스크립트 실행 타이밍 비교

| 속성 | 다운로드 | 실행 시점 | 블로킹 | 용도 |
|------|---------|---------|--------|------|
| 없음 | 동기 | 즉시 | ✅ (HTML 파싱 중단) | 레거시 코드 |
| `async` | 비동기 | 다운로드 완료 후 즉시 | ❌ | 독립적인 스크립트 |
| `defer` | 비동기 | HTML 파싱 완료 후 | ❌ | 메인 애플리케이션 |

### 실행 흐름 다이어그램

```
일반 <script>
  HTML 파싱
  → 🛑 스크립트 다운로드 & 실행
  → HTML 파싱 재개
  → DOM ready

<script async>
  HTML 파싱 (병렬로 스크립트 다운로드)
  → 🛑 다운로드 완료되면 즉시 실행
  → HTML 파싱 계속
  → DOM ready

<script defer>
  HTML 파싱 (병렬로 스크립트 다운로드)
  → HTML 파싱 완료
  → 스크립트 실행 (순서 보장)
  → DOM ready
```

---

## 2. defer와 JS 모듈 (ES Modules)

### ES 모듈의 기본 문법

```html
<script type="module" src="app.js"></script>
```

### defer의 자동 적용

**ES 모듈은 기본적으로 `defer` 속성과 동일하게 동작합니다.**

```html
<!-- 아래 두 가지는 거의 동일한 동작 -->
<script type="module" src="app.js"></script>
<script defer src="app.js"></script>

<!-- 단, type="module"이 자동 적용되는 것 -->
```

### 모듈 로딩의 특징

1. **비동기 로딩**: HTML 파싱을 블로킹하지 않음
2. **순서 보장**: 여러 모듈이 있을 때 선언 순서 유지
3. **의존성 해결**: 모듈의 import 경로 자동 처리
4. **격리된 스코프**: 각 모듈은 독립된 스코프 유지

---

## 3. 실제 예제

### 예제 1: 기본 defer 사용

```html
<!-- index.html -->
<!DOCTYPE html>
<html>
<head>
  <title>defer 예제</title>
  <!-- 1. 공통 유틸리티 -->
  <script src="utils.js" defer></script>
  
  <!-- 2. 메인 앱 (utils에 의존) -->
  <script src="app.js" defer></script>
  
  <!-- 3. 초기화 스크립트 (app에 의존) -->
  <script src="init.js" defer></script>
</head>
<body>
  <div id="app"></div>
</body>
</html>
```

**실행 순서 보장됨**: `utils.js` → `app.js` → `init.js`

### 예제 2: ES 모듈 사용

```html
<!-- index.html -->
<!DOCTYPE html>
<html>
<head>
  <title>ES Module 예제</title>
</head>
<body>
  <div id="app"></div>
  
  <!-- 모듈은 자동으로 defer처럼 동작 -->
  <script type="module" src="main.js"></script>
</body>
</html>
```

```javascript
// main.js
import { utils } from './utils.js';
import { initApp } from './app.js';

console.log('main.js 시작');
// HTML 파싱이 완료된 후 실행됨
```

```javascript
// utils.js
export const utils = {
  log: (msg) => console.log(`[Utils] ${msg}`),
  delay: (ms) => new Promise(resolve => setTimeout(resolve, ms))
};
```

```javascript
// app.js
import { utils } from './utils.js';

export function initApp() {
  utils.log('앱 초기화됨');
  const app = document.getElementById('app');
  app.innerHTML = '<h1>Hello Module</h1>';
}
```

### 예제 3: 모듈 + defer 혼합

```html
<!-- 권장하지 않음: 중복되는 속성 -->
<script type="module" src="app.js" defer></script>

<!-- 권장: 모듈은 defer 속성 필요 없음 -->
<script type="module" src="app.js"></script>
```

### 예제 4: 의존성이 있는 스크립트들

```html
<!DOCTYPE html>
<html>
<head>
  <title>의존성 관리</title>
</head>
<body>
  <h1>로그 영역</h1>
  <ul id="logs"></ul>

  <!-- ❌ 나쁜 예: 순서가 보장되지 않음 (async) -->
  <script src="logger.js" async></script>
  <script src="app.js" async></script>

  <!-- ✅ 좋은 예: 순서 보장됨 (defer) -->
  <script src="logger.js" defer></script>
  <script src="app.js" defer></script>

  <!-- ✅ 최고: 모듈 사용 -->
  <script type="module" src="main.js"></script>
</body>
</html>
```

```javascript
// logger.js (독립 스크립트로 export 필요 시)
window.Logger = {
  log: (msg) => {
    const ul = document.getElementById('logs');
    const li = document.createElement('li');
    li.textContent = msg;
    ul.appendChild(li);
  }
};
```

```javascript
// app.js (window.Logger 의존)
document.addEventListener('DOMContentLoaded', () => {
  Logger.log('App 로드됨');
  // 이제 Logger가 존재함 (defer 덕분에)
});
```

---

## 4. 로딩 타이밍 테스트

```html
<!DOCTYPE html>
<html>
<head>
  <title>타이밍 테스트</title>
</head>
<body>
  <h1 id="title">테스트</h1>

  <script>
    console.log('1. 인라인 스크립트 (즉시 실행)');
    console.log('title element 있나?', document.getElementById('title') !== null);
  </script>

  <script src="normal.js"></script>
  <!-- normal.js: "2. 일반 스크립트 (블로킹)" -->

  <script src="async.js" async></script>
  <!-- async.js: "3. async 스크립트 (비결정적)" -->

  <script src="deferred.js" defer></script>
  <!-- deferred.js: "4. defer 스크립트 (마지막)" -->

  <script>
    console.log('5. 마지막 인라인 스크립트');
    console.log('title element 있나?', document.getElementById('title') !== null);
  </script>
</body>
</html>
```

**예상 콘솔 출력** (defer 사용 시):
```
1. 인라인 스크립트 (즉시 실행)
title element 있나? true
2. 일반 스크립트 (블로킹)
5. 마지막 인라인 스크립트
title element 있나? true
4. defer 스크립트 (마지막)
```

---

## 5. 모듈 import 순서 보장

```javascript
// main.js
console.log('1. main.js 시작');

import { funcA } from './a.js';  // 자동으로 다운로드 & 파싱
import { funcB } from './b.js';

console.log('2. imports 완료');
funcA();
funcB();
```

```javascript
// a.js
console.log('A 모듈 로드됨');
export function funcA() {
  console.log('funcA 실행');
}
```

```javascript
// b.js
import { funcA } from './a.js';  // a.js가 먼저 로드됨

console.log('B 모듈 로드됨');
export function funcB() {
  console.log('funcB 실행');
  funcA();
}
```

**실행 순서**:
```
A 모듈 로드됨
B 모듈 로드됨
1. main.js 시작
2. imports 완료
funcA 실행
funcB 실행
funcA 실행
```


