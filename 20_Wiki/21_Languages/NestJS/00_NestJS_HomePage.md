

```
효율적이고 확장 가능한 Node.js 서버 백엔드 프레임워크
TypeScript 기본 지원 / 의존성 주입 / 마이크로서비스
```

> ⚠️ NestJS (서버 백엔드) ≠ Next.js (React 프론트엔드)

---

---

## Level 0. 개념 잡기

```
NestJS 가 뭔지, HTTP 와 REST API 기초
```

|노트|핵심 개념|
|---|---|
|[[NestJS_Concept]]|NestJS vs Next.js / HTTP / Method / Header / Body / REST API / 상태코드|
|[[NestJS_Install_Setup]]|설치 / CLI / 폴더 구조 / 각 파일 역할 / package.json / node_modules|
|[[NestJS_LifeCycle]] ⭐|REQUEST 흐름 / Middleware / Guard / Interceptor / Pipe / Controller / Service / Repository / Exception Filter|

---

---

## Level 1. 핵심 구성요소

```
NestJS 의 빌딩 블록
```

| 노트                            | 핵심 개념                                                              |
| ----------------------------- | ------------------------------------------------------------------ |
| [[NestJS_Module]] ⭐           | @Module / imports / controllers / providers / exports / 모듈화        |
| [[NestJS_Controller]] ⭐       | @Controller / @Get·Post·Put·Patch·Delete / @Param / @Body / @Query |
| [[NestJS_Service_Provider]] ⭐ | @Injectable / @InjectRepository / 비즈니스 로직 분리                       |
| [[NestJS_DI]]                 | 의존성 주입 / constructor 주입 / 제어의 역전(IoC)                              |

---

---

## Level 2. 요청 & 응답

```
HTTP 요청 처리 방법
```

|노트|핵심 개념|
|---|---|
|[[NestJS_Routing]]|라우팅 패턴 / 동적 파라미터 / 중첩 라우트|
|[[NestJS_DTO]] ⭐|DTO / class-validator / @IsString / @IsNumber / 유효성 검사|
|[[NestJS_Pipe]]|ValidationPipe / ParseIntPipe / 전역 파이프 설정|
|[[NestJS_Response]]|@HttpCode / @Res / 상태코드 / JSON 응답|

---

---

## Level 3. 미들웨어 & 인터셉터

|노트|핵심 개념|
|---|---|
|[[NestJS_Middleware]]|미들웨어 / NestMiddleware / 전역 / 특정 라우트 적용|
|[[NestJS_Guard]] ⭐|@UseGuards / CanActivate / 인증 가드 / JWT 가드|
|[[NestJS_Interceptor]]|로깅 / 응답 변환 / 에러 처리|
|[[NestJS_Exception]]|HttpException / NotFoundException / BadRequestException|

---

---

## Level 4. 데이터베이스

```
TypeORM / Prisma 연동
```

|노트|핵심 개념|
|---|---|
|[[NestJS_TypeORM]] ⭐|TypeORM / Entity / Repository / CRUD / 관계 설정|
|[[NestJS_Prisma]]|Prisma 연동 / schema / migrate / PrismaService|
|[[NestJS_PostgreSQL]]|PostgreSQL 연결 / .env / 커넥션 풀|

---

---

## Level 5. 인증 & 보안

|노트|핵심 개념|
|---|---|
|[[NestJS_Auth_JWT]] ⭐|JWT / Passport / AuthGuard / 로그인 / 토큰 발급|
|[[NestJS_Bcrypt]]|비밀번호 해싱 / bcrypt / salt|
|[[NestJS_CORS]]|CORS 설정 / 허용 Origin / 프론트 연동|

---

---

## 프로젝트 적용

|노트|설명|
|---|---|
|[[NestJS_REST_API_Pattern]]|CRUD 패턴 / 폴더 구조 / 모듈 분리 전략|
|[[NestJS_Env_Config]]|@nestjs/config / .env / ConfigService|
|[[NestJS_Deploy]]|Docker / PM2 / Vercel / Railway 배포|