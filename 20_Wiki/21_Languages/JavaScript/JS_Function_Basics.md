---
aliases:
  - 함수
  - 화살표 함수
  - Arrow Function
tags:
  - JavaScript
related:
  - "[[00_JS_HomePage]]"
  - "[[JS_Higher_Order]]"
  - "[[JS_Closure]]"
---

# JS_Function_Basics — 함수 기초

## 한 줄 요약

```
JS 에서 함수는 값처럼 다룰 수 있음
화살표 함수 → 콜백에서 가장 많이 씀
```

---

---

# ① 함수 선언 3가지

```javascript
// 1. function 선언문
function greet(name) {
  return `Hello, ${name}`;
}

// 2. 함수 표현식
const greet = function(name) {
  return `Hello, ${name}`;
};

// 3. 화살표 함수 ⭐️ (가장 많이 씀)
const greet = (name) => {
  return `Hello, ${name}`;
};

// 한 줄이면 중괄호 / return 생략
const greet = name => `Hello, ${name}`;
```

---

---

# ② 화살표 함수 ⭐️

```javascript
// 기본 형태
const add = (a, b) => a + b;

// 매개변수 1개면 () 생략 가능
const double = n => n * 2;

// 매개변수 없으면 () 필수
const hello = () => 'Hello!';

// 객체 반환할 때 () 감싸기
const makeObj = (id, title) => ({ id, title });
//                              ↑ () 없으면 {} 를 함수 블록으로 인식
```

## 콜백에서 화살표 함수 패턴 ⭐️

```javascript
const movies = [{ id: 1, title: '마이클' }, { id: 2, title: '악마' }];

// find
movies.find(m => m.id === 1);
//          ↑ 화살표 함수가 콜백으로 들어감

// filter
movies.filter(m => m.title.startsWith('마'));

// map
movies.map(m => m.title);

// 풀어서 쓰면:
movies.find(function(m) {
  return m.id === 1;
});
// 화살표 함수가 훨씬 간결
```

---

---

# ③ 단축 속성명 ⭐️

```javascript
const id = 3;
const title = '어벤져스';

// 기존 방식
const movie = { id: id, title: title };

// 단축 속성명 (변수명과 키가 같으면 생략)
const movie = { id, title };
// 완전히 동일

// NestJS 에서 사용
const movie: Movie = {
  id: this.idCounter++,
  title,     // title: title 생략
};
```

---

---

# ④ 나머지 매개변수 & 기본값

```javascript
// 기본값
function greet(name = '익명') {
  return `Hello, ${name}`;
}
greet();        // 'Hello, 익명'
greet('홍길동'); // 'Hello, 홍길동'

// 나머지 매개변수 (rest)
function sum(...nums) {
  return nums.reduce((acc, n) => acc + n, 0);
}
sum(1, 2, 3);   // 6
```

---

---

# ⑤ 구조분해 할당 ⭐️

## 배열 구조분해

```javascript
const arr = [1, 2, 3];

const [a, b, c] = arr;
// a=1, b=2, c=3

// 일부만 추출
const [first, , third] = arr;
// first=1, third=3
```

## 객체 구조분해

```javascript
const movie = { id: 1, title: '마이클', year: 2023 };

// 기본
const { id, title } = movie;
// id=1, title='마이클'

// 이름 변경
const { id: movieId } = movie;
// movieId=1

// 기본값
const { genre = '드라마' } = movie;
// genre='드라마' (movie에 없으므로)

// 함수 매개변수에서
function show({ id, title }) {
  console.log(id, title);
}
show(movie);
```

---

---

# ⑥ spread 연산자

```javascript
// 배열 복사
const original = [1, 2, 3];
const copy = [...original];

// 배열 합치기
const merged = [...arr1, ...arr2];

// 객체 복사
const movie = { id: 1, title: '마이클' };
const updated = { ...movie, title: '마이클2' };
// { id: 1, title: '마이클2' }

// PATCH 패턴 (특정 필드만 업데이트)
const patched = { ...movie, ...changes };
```

---

---

# 함수 선언 방식 비교

|방식|호이스팅|this 바인딩|주 용도|
|---|---|:-:|---|
|function 선언|✅ 됨|동적|일반 함수|
|function 표현식|❌|동적|변수 할당|
|화살표 함수|❌|상위 스코프|콜백 ⭐️|