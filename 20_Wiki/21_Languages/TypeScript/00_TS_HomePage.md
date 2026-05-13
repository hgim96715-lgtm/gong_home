
```
JavaScript 에 타입을 추가한 언어
컴파일 시 에러 발견 / 자동완성 / 팀 협업 안전성
```

---

---

## Level 0. 개념 & 설정

| 노트                   | 핵심 개념                                              |
| -------------------- | -------------------------------------------------- |
| [[TS_Concept]]       | TypeScript 란 / JS 와 차이 / 왜 쓰나 / 컴파일 vs 런타임         |
| [[TS_Install_Setup]] | tsc 설치 / tsconfig.json / strict 옵션 / ts-node / tsx |

---

---

## Level 1. 기본 타입

|노트|핵심 개념|
|---|---|
|[[TS_Basic_Types]]|string / number / boolean / null / undefined / any / unknown / never|
|[[TS_Array_Tuple]]|string[] / Array<T> / 튜플 [string, number] / readonly|
|[[TS_Type_Alias]] ⭐|type 키워드 / Union(|
|[[TS_Enum]]|enum / const enum / 숫자 enum / 문자열 enum|

---

---

## Level 2. 인터페이스 & 타입

| 노트                       | 핵심 개념                                             |
| ------------------------ | ------------------------------------------------- |
| [[TS_Interface]] ⭐       | interface / 선택적 속성 ? / readonly / extends / 병합 선언 |
| [[TS_Type_vs_Interface]] | type vs interface 차이 / 언제 뭘 쓰나                    |
| [[TS_Literal_Types]]     | 리터럴 타입 / const assertion / as const               |
| [[TS_Type_Guards]] ⭐     | typeof / instanceof / in / 사용자 정의 타입 가드           |

---

---

## Level 3. 함수 & 제네릭

|노트|핵심 개념|
|---|---|
|[[TS_Function_Types]]|매개변수 타입 / 반환 타입 / 선택적 매개변수 / 오버로드|
|[[TS_Generics]] ⭐|T / 제네릭 함수 / 제네릭 인터페이스 / extends 제약 / keyof|
|[[TS_Utility_Types]] ⭐|Partial / Required / Readonly / Pick / Omit / Record / ReturnType|

---

---

## Level 4. 클래스 & 고급 타입

|노트|핵심 개념|
|---|---|
|[[TS_Class]]|public / private / protected / readonly / abstract|
|[[TS_Mapped_Types]]|[K in keyof T] / 맵드 타입 / 조건부 타입|
|[[TS_Template_Literal]]|`${string}` / 템플릿 리터럴 타입|
|[[TS_Infer]]|infer / 조건부 타입 내 타입 추론|

---

---

## Level 5. 실전 패턴

|노트|핵심 개념|
|---|---|
|[[TS_API_Types]] ⭐|API 응답 타입 정의 / fetch 제네릭 / 에러 타입|
|[[TS_React_Types]]|FC / ReactNode / MouseEvent / ChangeEvent / Props 타입|
|[[TS_Module_Declaration]]|.d.ts / declare / 외부 라이브러리 타입|
|[[TS_Config_Options]]|strict / noImplicitAny / strictNullChecks / paths alias|

---

---

## 자주 하는 실수 & 패턴

| 노트                   | 핵심 개념                                                           |
| -------------------- | --------------------------------------------------------------- |
| [[TS_Common_Errors]] | Type 'X' is not assignable / Object possibly undefined / any 남발 |
| [[TS_Migration]]     | JS → TS 마이그레이션 / allowJs / 점진적 도입                               |