# 조건부 렌더링

React에서 조건부 렌더링은 JavaScript의 조건문과 동일하게 작성할 수 있다.

## if / else

```jsx
let content;
if (isLoggedIn) {
  content = <AdminPanel />;
} else {
  content = <LoginForm />;
}

return (
  <div>
    {content}
  </div>
);
```

## 삼항 연산자

더 간결하게 JSX 안에서 바로 조건을 처리할 수 있다.

```jsx
return (
  <div>
    {isLoggedIn ? <AdminPanel /> : <LoginForm />}
  </div>
);
```

## `&&` 연산자

else 분기가 필요 없을 때는 `&&`를 사용한다.

```jsx
return (
  <div>
    {isLoggedIn && <AdminPanel />}
  </div>
);
```

> `isLoggedIn`이 `true`이면 `<AdminPanel />`을 렌더링하고, `false`이면 아무것도 렌더링하지 않는다.

## 어트리뷰트에도 동일하게 적용

조건부 렌더링은 어트리뷰트에도 동일하게 동작한다.

```jsx
<img
  src={user.imageUrl}
  className={isActive ? 'avatar active' : 'avatar'}
/>
```
