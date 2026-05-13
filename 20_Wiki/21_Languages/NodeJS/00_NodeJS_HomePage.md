

```
브라우저 밖에서 JavaScript 를 실행하는 런타임
서버 / CLI 툴 / 데이터 파이프라인 자동화
```

---

---

## Level 0. 개념 잡기

|노트|핵심 개념|
|---|---|
|[[NodeJS_Concept]]|Node.js 란 / 이벤트 루프 / 논블로킹 I/O / npm vs yarn|
|[[NodeJS_Install_Setup]]|설치 / nvm / package.json / scripts / node_modules|
|[[NodeJS_REST_API]]|HTTP / Method / Header / Body / 상태코드 / REST 설계 원칙|

---

---

## Level 1. 기본 모듈

```
내장 모듈 — import 없이 바로 쓸 수 있는 것들
```

|노트|핵심 개념|
|---|---|
|[[NodeJS_File_System]] ⭐|fs.readFile / writeFile / existsSync / path.join / 동기 vs 비동기|
|[[NodeJS_Path_OS]]|path.join / path.resolve / __dirname / os.platform|
|[[NodeJS_HTTP]]|http.createServer / req / res / 포트 / 기본 서버|
|[[NodeJS_Process_Env]]|process.env / process.argv / process.exit / dotenv|

---

---

## Level 2. 비동기 패턴

```
Node.js 의 핵심 — 비동기 처리 방식
```

|노트|핵심 개념|
|---|---|
|[[NodeJS_Callback]]|콜백 함수 / 콜백 헬 / Error-first 패턴|
|[[NodeJS_Promise]] ⭐|Promise / .then .catch .finally / Promise.all / Promise.race|
|[[NodeJS_AsyncAwait]] ⭐|async/await / try-catch / 병렬 실행 / 에러 처리|
|[[NodeJS_EventEmitter]]|EventEmitter / on / emit / once / 커스텀 이벤트|

---

---

## Level 3. 패키지 & 모듈

|노트|핵심 개념|
|---|---|
|[[NodeJS_CommonJS_ESM]]|require vs import / module.exports / CJS vs ESM 차이|
|[[NodeJS_NPM]]|npm install / -D / -g / package-lock.json / npx|
|[[NodeJS_Useful_Packages]]|axios / dotenv / dayjs / lodash / uuid / chalk|

---

---

## Level 4. 서버 개발

```
Express 기반 백엔드 API 서버
```

|노트|핵심 개념|
|---|---|
|[[NodeJS_Express_Basics]] ⭐|app.get·post·put·delete / req.body / res.json / 포트|
|[[NodeJS_Express_Middleware]] ⭐|middleware / 순서 / morgan / cors / express.json|
|[[NodeJS_Express_Router]]|Router / 모듈화 / 컨트롤러 분리|
|[[NodeJS_Express_Error]]|에러 미들웨어 / 404 / try-catch / HTTP 상태코드|

---

---

## Level 5. 데이터베이스 연동

|노트|핵심 개념|
|---|---|
|[[NodeJS_PostgreSQL]] ⭐|pg / Pool / query / parameterized query / connection|
|[[NodeJS_Prisma]]|Prisma ORM / schema.prisma / migrate / CRUD|

---

---

## 프로젝트 적용

|노트|설명|
|---|---|
|[[NodeJS_REST_API]]|RESTful 설계 / CRUD 패턴 / 상태코드 규칙|
|[[NodeJS_Auth_JWT]]|JWT / bcrypt / 로그인 / 토큰 검증 미들웨어|
|[[NodeJS_CLI_Script]]|데이터 파이프라인 자동화 / 스케줄러 연동|