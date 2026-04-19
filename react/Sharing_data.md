# 컴포넌트 간에 데이터 공유하기

앞서 본 것처럼 각 컴포넌트는 독립적인 State를 가진다.
하지만 여러 컴포넌트가 **같은 State를 공유**해야 할 때는 State를 부모 컴포넌트로 올려야 한다.
이를 **State 끌어올리기(Lifting State Up)** 라고 한다.

## 문제 상황

```jsx
// 각 MyButton이 독립적인 count를 가짐 → 서로 다른 값
function MyApp() {
  return (
    <div>
      <MyButton />
      <MyButton />
    </div>
  );
}

function MyButton() {
  const [count, setCount] = useState(0);
  // ...
}
```

## State 끌어올리기

State를 부모로 올리고, 자식에게 **props**로 전달한다.

```jsx
export default function MyApp() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1);
  }

  return (
    <div>
      <h1>같이 업데이트되는 카운터</h1>
      <MyButton count={count} onClick={handleClick} />
      <MyButton count={count} onClick={handleClick} />
    </div>
  );
}

function MyButton({ count, onClick }) {
  return (
    <button onClick={onClick}>
      Clicked {count} times
    </button>
  );
}
```

> 부모가 State를 관리하고, 자식은 props로 받아서 표시만 한다.
> 어느 버튼을 클릭해도 부모의 `count`가 바뀌므로 두 버튼이 항상 같은 값을 보여준다.

## 데이터 흐름

```
MyApp (count, handleClick)
  ├── MyButton ← count, onClick (props)
  └── MyButton ← count, onClick (props)
```

React의 데이터는 항상 **부모 → 자식** 방향으로 흐른다.
