---
aliases:
  - NestJS 라이프사이클
  - 요청 흐름
  - Request Lifecycle
tags:
  - NestJS
related:
  - "[[00_NestJS_HomePage]]"
  - "[[NestJS_Middleware]]"
  - "[[NestJS_Guard]]"
  - "[[NestJS_Interceptor]]"
  - "[[NestJS_Pipe]]"
  - "[[NestJS_Exception]]"
---

# NestJS_LifeCycle — 요청 라이프사이클

## 한 줄 요약

```
HTTP 요청이 들어와서 응답이 나가기까지
NestJS 내부에서 거치는 처리 순서
```

---

---

# ① 전체 흐름 ⭐️

```
REQUEST 요청
    ↓
Middleware      요청 전처리 (로깅, CORS, 쿠키 파싱 등)
    ↓
Guard           인증/인가 확인 (이 요청 허용할지)
    ↓
Interceptor     요청 변환 / 로깅 (컨트롤러 실행 전)
    ↓
Pipe            데이터 유효성 검사 / 변환
    ↓
┌─────────────────────────┐
│  Controller             │  URL + 메서드 라우팅
│  Service                │  비즈니스 로직
│  Repository             │  DB 접근
└─────────────────────────┘
    ↓
Exception Filter    에러 발생 시 처리
    ↓
Interceptor         응답 변환 (컨트롤러 실행 후)
    ↓
RESPONSE 응답
```

![[NestJS_요청 라이프사이클.png]]


---

---

# ② 각 단계 역할 ⭐️

## Middleware

```
가장 먼저 실행
Express 미들웨어와 동일한 개념

역할:
  로깅 (morgan)
  CORS 처리
  쿠키 파싱
  요청 로깅

특징:
  Guard / Interceptor 와 다르게
  DI (의존성 주입) 미지원
```

→ [[NestJS_Middleware]] 참고

## Guard

```
인증 / 인가 담당
"이 요청을 허용할지 말지" 결정

역할:
  JWT 토큰 검증
  관리자 권한 확인
  로그인 여부 확인

반환값:
  true  → 다음 단계 진행
  false → 403 Forbidden

특징:
  Pipe 보다 먼저 실행
```

  → [[NestJS_Guard]] 참고

## Interceptor (요청 전)

```
컨트롤러 실행 전 & 후 모두 개입 가능

요청 전 역할:
  요청 로깅 (시작 시간 기록)
  요청 데이터 변환
  캐시 확인
```

→ [[NestJS_Interceptor]] 참고

## Pipe

```
컨트롤러에 도달하기 직전에 실행
데이터 유효성 검사 + 변환

역할:
  @Body() 데이터 타입 검증
  문자열 → 숫자 변환
  DTO 유효성 검사 (class-validator)

실패 시:
  400 Bad Request 자동 반환

```

→ [[NestJS_Pipe]] 참고

## Controller → Service → Repository

```
요청 로직 처리 부분 (핵심 비즈니스 영역)

Controller   URL 매핑 / 요청 받아서 Service 호출
Service      실제 비즈니스 로직 처리
Repository   DB 조회 / 저장 / 수정 / 삭제
```

→ [[NestJS_Controller]]
→ [[NestJS_Service_Provider]]

## Exception Filter

```
에러가 발생했을 때 잡아서 처리
try-catch 를 전역으로 적용하는 것과 유사

역할:
  에러 → 적절한 HTTP 응답으로 변환
  일관된 에러 응답 형식 유지

예시:
  NotFoundException → 404 응답
  UnauthorizedException → 401 응답
  커스텀 에러 → 커스텀 응답

```

→ [[NestJS_Exception]] 참고

## Interceptor (응답 후)

```
컨트롤러 실행이 끝나고 응답 반환 전

역할:
  응답 데이터 변환 (형식 통일)
  실행 시간 로깅 (종료 시간 기록)
  응답 캐싱

```

→ [[NestJS_Interceptor]] 참고

---

---

# ③ 실행 순서 정리

```
요청 들어올 때:
  Middleware → Guard → Interceptor(전) → Pipe → Controller

응답 나갈 때:
  Controller → Exception Filter → Interceptor(후) → 클라이언트

에러 발생 시:
  어느 단계에서든 에러 → Exception Filter 로 이동
```

---

---

# ④ 언제 뭘 쓰나 — 선택 가이드

|목적|사용할 것|
|---|---|
|로그인 여부 확인|Guard|
|요청 데이터 타입 검증|Pipe|
|요청/응답 로깅|Interceptor|
|응답 형식 통일|Interceptor|
|에러 처리|Exception Filter|
|CORS / 쿠키 처리|Middleware|
|비즈니스 로직|Service|
|DB 접근|Repository|

---

---
