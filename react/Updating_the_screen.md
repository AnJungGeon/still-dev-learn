# 화면 업데이트하기

컴포넌트가 특정 정보를 "기억"하여 표시해야 할 때 **State**를 사용한다.

## useState

```jsx
import { useState } from 'react';

function MyButton() {
  const [count, setCount] = useState(0);
  // ...
}
```

- `count` — 현재 State 값
- `setCount` — State를 업데이트하는 함수
- `useState(0)` — 초기값 `0`으로 설정

관례적으로 `[something, setSomething]` 형태로 이름을 짓는다.

## 예시: 클릭 카운터

```jsx
function MyButton() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1);
  }

  return (
    <button onClick={handleClick}>
      Clicked {count} times
    </button>
  );
}
```

`setCount(count + 1)`이 호출되면 React가 컴포넌트를 다시 렌더링하고, 새로운 `count` 값이 화면에 반영된다.

## State는 컴포넌트마다 독립적

같은 컴포넌트를 여러 번 렌더링하면 각 인스턴스는 **고유한 State**를 가진다.

```jsx
export default function MyApp() {
  return (
    <div>
      <h1>Counters that update separately</h1>
      <MyButton />
      <MyButton />
    </div>
  );
}
```

> 위 두 `<MyButton />`은 서로 다른 `count`를 가지며, 하나를 클릭해도 다른 하나에 영향을 주지 않는다.
