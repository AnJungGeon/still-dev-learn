# Hook 사용하기

`use`로 시작하는 함수를 **Hook**이라고 한다.
`useState`는 React가 제공하는 내장 Hook이다.

## Hook의 규칙

Hook은 **컴포넌트 최상위**에서만 호출할 수 있다.

```jsx
// 올바름 - 컴포넌트 최상위에서 호출
function MyComponent() {
  const [count, setCount] = useState(0);
  // ...
}

// 잘못됨 - 조건문 안에서 호출
function MyComponent() {
  if (someCondition) {
    const [count, setCount] = useState(0); // 오류
  }
}
```

> 조건문, 반복문, 중첩 함수 안에서는 Hook을 호출할 수 없다.

## 커스텀 Hook

여러 컴포넌트에서 반복되는 로직은 커스텀 Hook으로 만들 수 있다.
커스텀 Hook도 `use`로 시작해야 한다.

```jsx
function useWindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);

  useEffect(() => {
    const handleResize = () => setWidth(window.innerWidth);
    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  }, []);

  return width;
}

function MyComponent() {
  const width = useWindowWidth();
  return <p>창 너비: {width}</p>;
}
```
