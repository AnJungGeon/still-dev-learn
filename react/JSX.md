# JSX

위 Component 문서에서 본 마크업 문법을 JSX라고 한다. 선택사항이지만 편의성을 위해 React는 JSX를 사용한다.

## JSX의 정체

JSX는 **JavaScript의 확장 문법**이다. 브라우저는 JSX를 직접 이해하지 못하므로, Babel 같은 도구가 빌드 시점에 아래처럼 순수 JavaScript로 변환한다.

```jsx
// 우리가 작성하는 JSX
const element = <h1>안녕하세요</h1>;

// Babel이 변환한 실제 JavaScript
const element = React.createElement('h1', null, '안녕하세요');
```

즉, JSX는 JavaScript에 HTML과 유사한 마크업을 쓸 수 있게 해주는 **문법 확장**이다.

## JSX 규칙

JSX는 HTML보다 엄격하다.

### 1. 태그는 반드시 닫아야 한다

HTML에서는 `<br>` 처럼 열기만 해도 되지만, JSX에서는 `<br />` 처럼 반드시 닫아야 한다.

```jsx
// HTML에서는 OK, JSX에서는 오류
<br>
<img src="photo.png">

// JSX에서는 이렇게 써야 한다
<br />
<img src="photo.png" />
```

### 2. 하나의 부모 요소로 감싸야 한다

컴포넌트는 여러 개의 JSX 태그를 반환할 수 없다. 반드시 하나의 부모로 감싸야 한다.

```jsx
// 오류 - 루트 태그가 두 개
return (
  <h1>제목</h1>
  <p>내용</p>
);

// OK - div로 감싸기
return (
  <div>
    <h1>제목</h1>
    <p>내용</p>
  </div>
);

// OK - 빈 태그(Fragment)로 감싸기
return (
  <>
    <h1>제목</h1>
    <p>내용</p>
  </>
);
```

> `<div>` 대신 `<></>`(Fragment)를 쓰면 불필요한 DOM 요소 없이 묶을 수 있다.

## JavaScript 표현식 삽입

JSX 안에서 `{}`를 사용하면 JavaScript 표현식을 삽입할 수 있다.

```jsx
const name = '홍길동';
const age = 20;

return (
  <div>
    <p>이름: {name}</p>
    <p>나이: {age}</p>
    <p>내년 나이: {age + 1}</p>
  </div>
);
```

> `{}` 안에는 변수, 연산, 함수 호출 등 **값을 반환하는 표현식**이라면 무엇이든 들어갈 수 있다.
> 단, `if문`이나 `for문` 같은 **문(statement)** 은 직접 넣을 수 없다.

## HTML과 다른 속성명

JSX는 JavaScript 기반이기 때문에 일부 HTML 속성명이 다르다.

| HTML | JSX |
|------|-----|
| `class` | `className` |
| `for` | `htmlFor` |
| `onclick` | `onClick` |
| `onchange` | `onChange` |

`class`와 `for`는 JavaScript의 예약어이기 때문에 다른 이름을 사용한다.
그 외 이벤트 속성들은 camelCase로 바뀐다.

```jsx
// HTML
<label class="label" for="input-id">이름</label>
<input class="input" type="text" id="input-id" onchange="handleChange()" />

// JSX
<label className="label" htmlFor="input-id">이름</label>
<input className="input" type="text" id="input-id" onChange={handleChange} />
```
