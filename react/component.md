# 컴포넌트 (Component)

React 앱은 **컴포넌트**로 구성된다.

컴포넌트는 고유한 로직과 모양을 가진 UI의 일부이다.
크기는 작은 버튼 하나일 수도 있고, 전체 페이지일 수도 있다.

## 컴포넌트 선언

```jsx
function MyButton() {
  return (
    <button>버튼이에요</button>
  );
}
```

## export default

`export default`는 JavaScript의 모듈 문법으로, **이 파일에서 기본으로 내보낼 것**을 지정한다.

```jsx
// MyApp.jsx
export default function MyApp() { ... }

// 다른 파일에서 불러올 때
import MyApp from './MyApp';
```

- `export default`가 있어야 다른 파일에서 `import`할 수 있고, 이것이 **컴포넌트를 분리해서 재사용**하는 기반이 된다.

- JSP의 `<jsp:include>`처럼 다른 파일을 가져와 조합하는 방식과 유사하다. 다만 JSP는 서버에서 HTML을 조합하고, React는 클라이언트에서 컴포넌트를 조합한다.

- 바닐라 JS의 `innerHTML`로 HTML을 동적으로 끼워넣는 방식과도 유사하다. 다만 `innerHTML`은 DOM을 통째로 교체하지만, React는 상태 변화가 생기면 **변경된 부분만** 효율적으로 업데이트한다. (Virtual DOM)

- 파일 하나에 `export default`는 **하나만** 사용할 수 있다.

- `default` 없이 `export`만 쓰면 named export가 되어 불러올 때 `{}` 가 필요하다.

```jsx
// named export
export function MyButton() { ... }

// 불러올 때
import { MyButton } from './MyButton';
```

## 컴포넌트 중첩

`MyButton`을 선언했으므로 다른 컴포넌트 안에 중첩할 수 있다.

```jsx
export default function MyApp() {
  return (
    <div>
      <h1>Welcome to my app</h1>
      <MyButton />
    </div>
  );
}
```

> React 컴포넌트는 항상 **대문자**로 시작해야 한다.
> HTML 태그는 소문자, React 컴포넌트는 대문자로 구분한다.
