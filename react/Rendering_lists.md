# 리스트 렌더링하기

데이터 목록을 렌더링할 때는 JavaScript의 `map()`을 사용한다.

```jsx
const products = [
  { title: 'Cabbage', id: 1 },
  { title: 'Garlic', id: 2 },
  { title: 'Apple', id: 3 },
];

const listItems = products.map(product =>
  <li key={product.id}>
    {product.title}
  </li>
);

return <ul>{listItems}</ul>;
```

## key

리스트를 렌더링할 때는 각 항목에 `key`를 지정해야 한다.
`key`는 React가 항목을 식별하는 데 사용하며, 삽입·삭제·정렬 시 올바르게 업데이트하기 위해 필요하다.

```jsx
<li key={product.id}>
```

> `key`는 형제 항목 사이에서 고유해야 한다.
> 보통 데이터베이스의 ID를 사용하며, 배열 인덱스는 순서가 바뀔 수 있어 권장하지 않는다.

## filter()와 함께 사용

조건에 맞는 항목만 렌더링할 때는 `filter()`를 먼저 사용한다.

```jsx
const products = [
  { title: 'Cabbage', isFruit: false, id: 1 },
  { title: 'Garlic', isFruit: false, id: 2 },
  { title: 'Apple', isFruit: true, id: 3 },
];

const fruitItems = products
  .filter(product => product.isFruit)
  .map(product =>
    <li key={product.id}>
      {product.title}
    </li>
  );

return <ul>{fruitItems}</ul>;
```
