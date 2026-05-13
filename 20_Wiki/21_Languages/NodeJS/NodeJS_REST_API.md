---
aliases:
  - HTTP
  - REST API
  - 상태코드
tags:
  - NodeJS
related:
  - "[[00_NodeJS_HomePage]]"
  - "[[NestJS_Concept]]"
---

# NodeJS_REST_API — HTTP & REST API

## 한 줄 요약

```
HTTP = 클라이언트와 서버가 통신하는 방법
REST = HTTP 를 설계된 의도대로 규격화한 API 설계 방식
```

---

---

# ① HTTP 요청 구성요소 ⭐️

```
URL     요청 주소
Method  요청 종류 (GET / POST / PUT / PATCH / DELETE)
Header  요청 메타데이터
Body    실제 전달 데이터
```

## URL 구성

```
https://api.example.com/users/123?sort=desc

https://         프로토콜
api.example.com  호스트 (서버 주소)
/users/123       경로 (리소스)
?sort=desc       쿼리 파라미터
```

## Method ⭐️

|Method|용도|Body|
|---|---|:-:|
|`GET`|데이터 조회|❌|
|`POST`|데이터 생성|✅|
|`PUT`|전체 업데이트 또는 생성|✅|
|`PATCH`|부분 업데이트|✅|
|`DELETE`|삭제|❌|

```
같은 URL 에 여러 Method 가능:
  GET  /posts  → 목록 조회
  POST /posts  → 게시글 생성

⚠️ Method 는 정해진 목적대로 사용
  어길 수 있지만 → 팀 혼란 야기 → 절대 금지
```

## Header

```
요청·응답 자체에 대한 메타데이터
Key / Value 형태 (둘 다 String)

자주 쓰는 Header:
  Content-Type   : application/json
  Authorization  : Bearer <JWT토큰>
  Cookie         : 인증 쿠키

라이브러리가 자동 생성하는 경우가 많음
```

## Body

```
요청 수행에 필요한 실제 데이터
주로 JSON 구조 사용

POST /posts Body 예시:
  { "title": "제목", "content": "내용" }

Header vs Body:
  Header = 요청 자체의 정보 (메타데이터)
  Body   = 요청 수행에 필요한 데이터
```

---

---

# ② JSON

```
요청·응답 body 에 사용하는 데이터 구조
Key(String) / Value(모든 타입) 쌍

보낼 때: JSON.stringify(obj)  → String
받을 때: JSON.parse(str)      → 객체
```

```json
{
  "title": "블로그 글",
  "views": 100,
  "published": true,
  "tags": ["nodejs", "http"]
}
```

---

---

# ③ HTTP 상태코드 ⭐️

|범위|의미|주요 코드|
|---|---|---|
|`2xx`|성공|200 OK / 201 Created / 204 No Content|
|`3xx`|리다이렉트|301 Moved Permanently / 302 Found|
|`4xx`|클라이언트 에러|400 Bad Request / 401 Unauthorized / 403 Forbidden / 404 Not Found|
|`5xx`|서버 에러|500 Internal Server Error / 503 Service Unavailable|

```
자주 쓰는 것:
  200  성공 (조회)
  201  생성 성공
  204  성공 + 응답 데이터 없음 (DELETE)
  400  잘못된 요청 (유효성 검사 실패)
  401  인증 필요 (로그인 안 됨)
  403  권한 없음 (로그인은 됐지만 접근 불가)
  404  리소스 없음
  500  서버 내부 에러
```

---

---

# ④ REST API 설계 ⭐️

## REST 원칙

```
균일한 인터페이스   표준 형식 (HTTP 메서드 + URL)
무상태(Stateless)  각 요청은 독립적
캐시 가능성        공통 응답 캐싱 가능
계층화 시스템      서버 여러 계층 구성 가능
```

## URL 설계 규칙

```
리소스(명사) 기반 / 동사 금지

✅ 올바른 URL:
  GET    /posts        게시글 목록
  GET    /posts/1      게시글 1번
  POST   /posts        생성
  PUT    /posts/1      전체 수정
  PATCH  /posts/1      부분 수정
  DELETE /posts/1      삭제

❌ 잘못된 URL:
  GET /getPosts       동사 사용
  POST /createPost    동사 사용
  GET /posts/delete/1 경로에 동작 포함
```

## 중첩 리소스

```
GET  /users/1/posts     사용자 1의 게시글 목록
POST /users/1/posts     사용자 1의 게시글 생성
GET  /users/1/posts/5   사용자 1의 게시글 5번
```