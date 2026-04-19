# 데이터 표시하기

JSX를 사용하면 JavaScript 안에 마크업을 삽입할 수 있다.
`{}`를 사용하면 변수나 표현식을 JSX 안에서 출력할 수 있다.

```jsx
return (
  <h1>
    {user.name}
  </h1>
);
```

## 어트리뷰트에서 `{}` 사용

JSX 어트리뷰트에서도 따옴표 대신 `{}`를 사용해 JavaScript 값을 전달할 수 있다.

```jsx
return (
  <img
    className="avatar"
    src={user.imageUrl}
  />
);
```

- `className="avatar"` — 문자열 `"avatar"`를 CSS 클래스로 전달
- `src={user.imageUrl}` — JavaScript 변수 `user.imageUrl`의 값을 전달

## 표현식 삽입

`{}` 안에는 변수뿐만 아니라 문자열 연결, 연산 등 값을 반환하는 표현식이라면 무엇이든 넣을 수 있다.

```jsx
const user = {
  name: 'Hedy Lamarr',
  imageUrl: 'https://i.imgur.com/yXOvdOSs.jpg',
  imageSize: 90,
};

export default function Profile() {
  return (
    <>
      <h1>{user.name}</h1>
      <img
        className="avatar"
        src={user.imageUrl}
        alt={'Photo of ' + user.name}
        style={{
          width: user.imageSize,
          height: user.imageSize,
        }}
      />
    </>
  );
}
```

> `style={{}}` 은 특별한 문법이 아니다. `style={ }` JSX 중괄호 안에 일반 JavaScript 객체 `{}`를 전달하는 것이다.
> 스타일 값이 JavaScript 변수에 의존할 때 `style` 어트리뷰트를 활용한다.
