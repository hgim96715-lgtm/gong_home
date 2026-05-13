---
aliases:
  - NestJs 설치
  - Nest New
  - NestJS CLI
tags:
  - NestJS
related:
  - "[[00_NestJS_HomePage]]"
  - "[[NestJS_Concept]]"
---
# NestJS_Install_Setup — 설치 & 프로젝트 시작

---

---

# ① Node.js 설치 확인

```bash
node --version
# v22.22.2

npm --version
# 10.9.7
```

```
Node.js 가 없으면:
  https://nodejs.org 에서 LTS 버전 설치
  설치하면 npm 자동으로 같이 설치됨
```

---

---

# ② pnpm 설치 (권장 패키지 매니저)

```bash
sudo npm install -g pnpm

pnpm --version
```

```
pnpm 쓰는 이유:
  npm / yarn 보다 빠르고 디스크 절약
  NestJS CLI 에서 pnpm 선택 가능
```

---

---

# ③ NestJS CLI 설치

```bash
sudo npm install -g @nestjs/cli

nest --version
```

---

---

# ④ 새 프로젝트 생성

## 방법 1 — 새 폴더로 시작

```bash
nest new hello-world
# 패키지 매니저 선택 → pnpm 선택
# ✅ pnpm

cd hello-world
```

## 방법 2 — 기존 Git 폴더에 생성 ⭐️

```bash
# 이미 만들어둔 폴더로 이동
cd nestjs-content-data-pipeline

# 현재 폴더에 NestJS 프로젝트 생성 (. = 현재 위치)
nest new .
# 패키지 매니저 선택 → pnpm 선택
```

```
nest new 폴더명  → 새 폴더 생성하며 시작
nest new .      → 현재 폴더에 바로 생성
                  Git 저장소 먼저 만들고 진행할 때
```

---

---

# ⑤ 프로젝트 폴더 구조

```
hello-world/
├── src/                     핵심 소스 코드
│   ├── main.ts              진입점 (서버 시작)
│   ├── app.module.ts        루트 모듈 (전체 앱 조립)
│   ├── app.controller.ts    컨트롤러 (요청 처리 / 라우팅)
│   ├── app.controller.spec.ts  컨트롤러 테스트
│   └── app.service.ts       서비스 (비즈니스 로직)
├── test/                    e2e 테스트
├── node_modules/            설치된 패키지 (건드리지 않음)
├── package.json             프로젝트 설정 / 의존성 목록
├── package-lock.json        의존성 버전 고정 파일
├── tsconfig.json            TypeScript 컴파일 설정
├── tsconfig.build.json      빌드용 TS 설정
├── nest-cli.json            NestJS CLI 설정
└── .eslintrc.js             코드 스타일 규칙
```

## src/ 각 파일 역할 ⭐️

```
main.ts
  앱의 시작점 (Entry Point)
  NestFactory.create(AppModule) 로 앱 생성
  포트 지정 후 서버 시작

app.module.ts
  앱 전체를 조립하는 루트 모듈
  어떤 컨트롤러/서비스를 쓸지 등록
  다른 모듈을 imports 에 추가

app.controller.ts
  URL 경로와 HTTP 메서드 처리
  @Get('/') → 요청 받아서 서비스로 넘김
  비즈니스 로직은 직접 처리 안 함

app.service.ts
  실제 비즈니스 로직 담당
  DB 조회, 계산, 데이터 가공 등
  컨트롤러에서 호출해서 사용
```

## main.ts — 서버 진입점

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);  // 포트 3000
}
bootstrap();
```

---

---

# ⑥ package.json 분석 ⭐️

```json
{
  "name": "hello-world",
  "version": "0.0.1",
  "scripts": {
    "start": "nest start",
    "start:dev": "nest start --watch",
    "start:prod": "node dist/main",
    "build": "nest build",
    "test": "jest"
  },
  "dependencies": {
    "@nestjs/common": "^10.0.0",
    "@nestjs/core": "^10.0.0",
    "@nestjs/platform-express": "^10.0.0",
    "reflect-metadata": "^0.1.13",
    "rxjs": "^7.8.1"
  },
  "devDependencies": {
    "@nestjs/cli": "^10.0.0",
    "@nestjs/testing": "^10.0.0",
    "typescript": "^5.1.3",
    "jest": "^29.5.0"
  }
}
```

```
scripts:
  start:dev    개발 중 사용 → --watch 로 파일 변경 감지 자동 재시작
  start:prod   배포 후 사용 → 빌드된 dist/main.js 실행
  build        TypeScript → JavaScript 컴파일 (dist/ 폴더 생성)
  test         테스트 실행

dependencies vs devDependencies:
  dependencies    실제 서버 실행에 필요한 패키지
  devDependencies 개발할 때만 필요한 패키지 (테스트/TypeScript/CLI)
  → 배포 시 devDependencies 는 설치 안 해도 됨

주요 패키지:
  @nestjs/common   NestJS 핵심 데코레이터 (@Get, @Post, @Injectable...)
  @nestjs/core     NestJS 프레임워크 엔진
  @nestjs/platform-express  내부적으로 Express 사용
  reflect-metadata TypeScript 데코레이터 동작에 필요
  rxjs             비동기 처리 라이브러리
```

---

---

# ⑦ node_modules / 의존성 관리

```
node_modules/
  package.json 에 적힌 패키지가 실제로 설치되는 폴더
  → pnpm install 실행 시 자동 생성
  → 용량이 매우 크고 (수백 MB) Git 에 올리지 않음 (.gitignore 에 포함)

package-lock.json (npm) / pnpm-lock.yaml (pnpm):
  의존성의 정확한 버전을 고정
  팀원 전체가 동일한 버전 설치하게 보장
  → Git 에 반드시 포함
```

```bash
# 의존성 설치 (package.json 기준)
pnpm install

# 새 패키지 추가
pnpm add @nestjs/config

# 개발용 패키지 추가
pnpm add -D @types/node

# node_modules 삭제 후 재설치 (문제 생길 때)
rm -rf node_modules
pnpm install
```

```
node_modules 가 없을 때:
  Git clone 후 → pnpm install 실행
  node_modules 는 package.json 에서 언제든 재생성 가능
  → Git 에 올릴 필요 없음 / .gitignore 에 포함
```

---

---

# ⑥ 서버 실행

```bash
# 개발 모드 (파일 변경 시 자동 재시작)
pnpm run start:dev

# 기본 실행
pnpm run start

# 빌드 후 실행 (프로덕션)
pnpm run build
pnpm run start:prod
```

```
서버 실행 후:
  http://localhost:3000 접속
  "Hello World!" 응답 확인
```

---

---

# ⑦ Postman — API 테스트 도구

```
Postman = GUI 로 HTTP 요청을 테스트하는 도구
  GET / POST / PUT / DELETE 요청 직접 전송
  Header / Body 직접 입력
  응답 확인

다운로드: https://www.postman.com/downloads/
```

```
사용 흐름:
  1. 서버 실행 (pnpm run start:dev)
  2. Postman 에서 GET http://localhost:3000 입력
  3. Send 클릭 → 응답 확인
```

---

---

# 명령어 한눈에

|명령어|역할|
|---|---|
|`nest new 프로젝트명`|새 프로젝트 생성|
|`nest new .`|현재 폴더에 생성|
|`pnpm run start:dev`|개발 서버 실행|
|`nest generate module 이름`|모듈 생성|
|`nest generate controller 이름`|컨트롤러 생성|
|`nest generate service 이름`|서비스 생성|
|`nest g resource 이름`|모듈+컨트롤러+서비스 한번에|