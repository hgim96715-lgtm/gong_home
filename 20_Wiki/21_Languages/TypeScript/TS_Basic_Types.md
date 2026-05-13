---
aliases:
  - TypeScript 기본 타입
  - 타입 어노테이션
tags:
  - TypeScript
related:
  - "[[00_TS_HomePage]]"
  - "[[TS_Interface]]"
  - "[[TS_Type_Alias]]"
---

# TS_Basic_Types — 기본 타입

## 한 줄 요약

```
변수/매개변수/반환값 뒤에 : 타입 으로 타입 지정
컴파일 시 타입 불일치 에러 발견
```

---

---

# ① 타입 어노테이션 기본 문법

```typescript
// 변수
let name: string = '홍길동';
let age: number = 30;
let isActive: boolean = true;

// 함수 매개변수 + 반환값
function greet(name: string): string {
  return `Hello, ${name}`;
}

// NestJS 에서 자주 쓰는 패턴
getMovie(@Param('id') id: string): any { }
//                        ↑ string   ↑ any
```

---

---

# ② 원시 타입

```typescript
let str: string = '문자열';
let num: number = 42;
let bool: boolean = true;

let nothing: null = null;
let notDefined: undefined = undefined;
```

---

---

# ③ any / unknown / never ⭐️

```typescript
// any — 타입 검사 포기 (가급적 사용 금지)
let anything: any = '문자열';
anything = 42;        // OK
anything = true;      // OK
// TS 의 의미가 없어짐 → 꼭 필요할 때만 사용

// unknown — any 의 안전한 버전
let val: unknown = '문자열';
val.toUpperCase();         // ❌ 에러 (타입 확인 없이 사용 불가)
if (typeof val === 'string') {
  val.toUpperCase();       // ✅ 타입 확인 후 사용
}

// never — 절대 발생하지 않는 타입
function throwError(msg: string): never {
  throw new Error(msg);    // 반환되지 않는 함수
}
```

```
any vs unknown:
  any     → 타입 검사 완전 포기 ⚠️
  unknown → 사용 전 타입 확인 강제 ✅
  → unknown 이 더 안전 / any 는 최후 수단
```

---

---

# ④ 배열 타입

```typescript
// 방법 1
let nums: number[] = [1, 2, 3];
let strs: string[] = ['a', 'b', 'c'];

// 방법 2 (제네릭)
let nums: Array<number> = [1, 2, 3];

// NestJS 에서
private movies: Movie[] = [];
//              ↑ Movie 타입의 배열
```

---

---

# ⑤ 타입 추론

```typescript
// 타입 어노테이션 없어도 TS 가 자동으로 추론
let name = '홍길동';      // string 으로 추론
let age = 30;             // number 로 추론
let flag = true;          // boolean 으로 추론

name = 42;   // ❌ 에러: string 에 number 할당 불가

// 명시적으로 써야 할 때:
//  함수 매개변수 (자동 추론 안 됨)
//  반환 타입 명시하고 싶을 때
//  복잡한 객체
```

---

---

# ⑥ Union 타입 | ⭐️

```typescript
// 여러 타입 중 하나
let id: string | number;
id = '1';    // OK
id = 1;      // OK
id = true;   // ❌ 에러

// 함수에서
function printId(id: string | number) {
  if (typeof id === 'string') {
    console.log(id.toUpperCase());
  } else {
    console.log(id * 2);
  }
}
```

---

---

# ⑦ 타입 변환 (NestJS 실전)

```typescript
// URL 파라미터는 항상 string
@Param('id') id: string

// 숫자로 비교할 때 변환 필요
const movie = movies.find(m => m.id === parseInt(id));
const movie = movies.find(m => m.id === +id);       // 단축형
const movie = movies.find(m => m.id === Number(id));

// 실수:
movies.find(m => m.id === id)  // ❌ number === string → 항상 false
```

---

---

# 타입 한눈에

|타입|예시 값|용도|
|---|---|---|
|`string`|`'hello'`|문자열|
|`number`|`42`, `3.14`|숫자|
|`boolean`|`true`, `false`|참/거짓|
|`null`|`null`|값 없음|
|`undefined`|`undefined`|미정의|
|`any`|뭐든지|⚠️ 최후 수단|
|`unknown`|뭐든지|any 의 안전 버전|
|`never`|(없음)|반환 안 하는 함수|
|`string[]`|`['a', 'b']`|문자열 배열|
|`string \| number`|`'1'` or `1`|둘 중 하나|