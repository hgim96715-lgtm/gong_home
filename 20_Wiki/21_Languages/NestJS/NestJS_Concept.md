---
aliases:
  - NextJS
  - 서버 프레임워크
  - NodeJS
  - backend
tags:
  - NestJS
related:
  - "[[00_NestJS_HomePage]]"
  - "[[NodeJS_Concept]]"
  - "[[NodeJS_REST_API]]"
---

# NextJS_Concept — NestJS 개념

## ⚠️ NestJS vs Next.js 헷갈리지 않기

```
NestJS  (Nest)  = Node.js 서버 백엔드 프레임워크
                  Express / Fastify 기반 / DI / 마이크로서비스
                  → 서버 API 만들 때 사용

Next.js (Next)  = React 기반 풀스택 프레임워크
                  SSR / SSG / 파일 기반 라우팅
                  → 웹 프론트엔드 + API 만들 때 사용
```

---

---

# ① NestJS — Node.js 서버 프레임워크

## 한 줄 요약

```
효율적이고 확장 가능한 Node.js 서버 프레임워크
TypeScript 기본 지원 / 의존성 주입 / 마이크로서비스
```

## 특징

```
모듈화된 아키텍처       기능별 모듈 분리 → 유지보수 편함
TypeScript 기본 지원   강력한 타이핑 / 팀 협업 안전성
의존성 주입 (DI)       프레임워크 자체 DI 시스템 제공
REST API + GraphQL    둘 다 네이티브 지원
마이크로서비스 설계     분산 시스템에 강력
```

## 내부 구조

```
기본: Express 기반
설정 변경: Fastify 로 교체 가능 (더 빠름)
```

## NestJS vs Express 비교

|항목|NestJS|Express|
|---|---|---|
|TypeScript 기본 지원|✅|❌|
|자체 아키텍처 / 모듈|✅|❌|
|의존성 주입 (DI)|✅|❌|
|마이크로서비스 기능|✅|❌|
|학습 난이도|높음|낮음|
|자유도|낮음|높음|

---

---

# ② HTTP & REST API


NestJS 로 API 를 만들기 전에 HTTP 와 REST 를 이해해야 함
자세한 내용 → [[NodeJS_REST_API]]

핵심 요약:
  요청 구성: URL / Method / Header / Body
  Method:   GET(조회) / POST(생성) / PUT(수정) / DELETE(삭제)
  상태코드:  200 성공 / 201 생성 / 400 잘못된요청 / 404 없음 / 500 서버에러
  REST:     HTTP 를 규격화한 API 설계 방식 / 리소스(명사) 기반 URL
