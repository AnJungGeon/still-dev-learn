# 이벤트에 응답하기

컴포넌트 내부에 이벤트 핸들러 함수를 선언하여 이벤트에 응답할 수 있다.

```jsx
function MyButton() {
  function handleClick() {
    alert('You clicked me!');
  }

  return (
    <button onClick={handleClick}>
      Click me
    </button>
  );
}
```

> `onClick={handleClick}`에 소괄호 `()`가 없는 것을 주목하자.
> 함수를 **호출**하는 것이 아니라 **전달**만 하면 된다.
> React가 사용자가 버튼을 클릭할 때 직접 호출한다.

| 작성 방법 | 의미 |
|---|---|
| `onClick={handleClick}` | 클릭 시 `handleClick` 실행 (올바름) |
| `onClick={handleClick()}` | 렌더링 즉시 `handleClick` 실행 (잘못됨) |
