---
aliases:
  - 배열 메서드
  - Array Methods
tags:
  - JavaScript
related:
  - "[[00_JS_HomePage]]"
  - "[[JS_Function_Basics]]"
  - "[[JS_Array_Tricks]]"
---

# JS_Array_Methods — 배열 메서드

## 한 줄 요약

```
배열을 다루는 내장 메서드
NestJS / 데이터 처리에서 매일 사용
```

---

---

# ① 탐색 — find / findIndex / filter ⭐️

## find — 조건에 맞는 첫 번째 항목

```javascript
const movies = [
  { id: 1, title: '마이클' },
  { id: 2, title: '악마는 프라다를 입는다' },
];

// 조건에 맞는 첫 번째 항목 반환
const movie = movies.find(m => m.id === 1);
// { id: 1, title: '마이클' }

// 없으면 undefined 반환
const none = movies.find(m => m.id === 99);
// undefined → !none 으로 체크
```

```
find vs filter:
  find   → 첫 번째 1개만 반환 / 없으면 undefined
  filter → 조건 맞는 전체 배열 반환 / 없으면 빈 배열 []
```

## findIndex — 인덱스 찾기

```javascript
const index = movies.findIndex(m => m.id === 2);
// 1 (배열에서 두 번째 = index 1)

// 없으면 -1 반환
const notFound = movies.findIndex(m => m.id === 99);
// -1
if (index === -1) { /* 없음 */ }
```

```
find vs findIndex:
  find       → 항목 자체 반환 (수정/확인 목적)
  findIndex  → 위치(숫자) 반환 (삭제 목적)

삭제할 때 findIndex 쓰는 이유:
  splice(index, 1) 로 삭제하려면 인덱스가 필요
```

## filter — 조건에 맞는 전체

```javascript
// 조건에 맞는 전체 배열 반환
const result = movies.filter(m => m.title.startsWith('마'));
// [{ id: 1, title: '마이클' }]

// 없으면 빈 배열
const empty = movies.filter(m => m.id === 99);
// []
```

---

---

# ② 추가 / 삭제 — push / splice / pop

## push — 맨 뒤에 추가

```javascript
movies.push({ id: 3, title: '어벤져스' });
// movies 에 추가됨 (원본 변경)

// 반환값: 추가 후 배열 길이
const newLength = movies.push({ id: 4, title: '겨울왕국' });
// 4
```

## splice — 인덱스 위치에서 삭제/추가

```javascript
// splice(시작인덱스, 삭제개수)
movies.splice(1, 1);    // index 1 위치에서 1개 삭제

// 삭제 패턴 (NestJS DELETE)
const index = movies.findIndex(m => m.id === +id);
if (index === -1) throw new NotFoundException(...);
movies.splice(index, 1);   // 해당 위치 삭제
```

```
splice vs slice:
  splice  원본 배열 변경 (삭제/추가)
  slice   원본 유지, 잘라낸 새 배열 반환

splice(index, 1) 패턴:
  findIndex 로 위치 찾고
  splice 로 그 위치 삭제
  → DELETE API 구현의 핵심
```

## pop / shift

```javascript
movies.pop()     // 맨 뒤 삭제 + 반환
movies.shift()   // 맨 앞 삭제 + 반환
```

---

---

# ③ 변환 — map / sort / flat

## map — 각 항목을 변환

```javascript
const titles = movies.map(m => m.title);
// ['마이클', '악마는 프라다를 입는다']

const ids = movies.map(m => m.id);
// [1, 2]

// 객체 변환
const modified = movies.map(m => ({
  ...m,           // 기존 속성 복사
  title: m.title.toUpperCase()  // title 변환
}));
```

## sort — 정렬

```javascript
const nums = [3, 1, 4, 1, 5];

// 오름차순
nums.sort((a, b) => a - b);   // [1, 1, 3, 4, 5]

// 내림차순
nums.sort((a, b) => b - a);   // [5, 4, 3, 1, 1]

// 문자열 정렬
movies.sort((a, b) => a.title.localeCompare(b.title));
```

---

---

# ④ 집계 — reduce / some / every / includes

## reduce — 누적 계산

```javascript
const nums = [1, 2, 3, 4, 5];

const sum = nums.reduce((acc, cur) => acc + cur, 0);
// 15

// 객체 배열 합산
const totalId = movies.reduce((acc, m) => acc + m.id, 0);
// 3 (1 + 2)
```

## some / every

```javascript
// some: 하나라도 조건 만족?
movies.some(m => m.id === 1);   // true
movies.some(m => m.id === 99);  // false

// every: 전부 조건 만족?
movies.every(m => m.id > 0);    // true
movies.every(m => m.id > 1);    // false
```

## includes

```javascript
const arr = [1, 2, 3];
arr.includes(2);   // true
arr.includes(99);  // false
```

---

---

# 메서드 한눈에

|메서드|반환|원본 변경|주 용도|
|---|---|:-:|---|
|`find(fn)`|항목 or undefined|❌|단건 조회|
|`findIndex(fn)`|숫자 or -1|❌|삭제 전 위치 확인|
|`filter(fn)`|배열|❌|조건 검색|
|`map(fn)`|배열|❌|데이터 변환|
|`push(항목)`|길이|✅|추가|
|`pop()`|항목|✅|마지막 삭제|
|`splice(i, n)`|삭제된 배열|✅|중간 삭제|
|`slice(s, e)`|배열|❌|자르기|
|`sort(fn)`|배열|✅|정렬|
|`reduce(fn, init)`|단일값|❌|누적 계산|
|`some(fn)`|boolean|❌|하나라도|
|`every(fn)`|boolean|❌|전부|
|`includes(값)`|boolean|❌|포함 여부|