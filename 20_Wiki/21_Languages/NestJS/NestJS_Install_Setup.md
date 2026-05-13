---
aliases:
  - NestJs 설치
  - Nest New
  - NestJS CLI
tags:
  - NestJS
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

![[스크린샷 2026-05-12 오후 3.38.45.png]]
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
├── src/
│   ├── app.controller.ts    컨트롤러 (요청 처리)
│   ├── app.controller.spec.ts  테스트
│   ├── app.module.ts        루트 모듈
│   ├── app.service.ts       서비스 (비즈니스 로직)
│   └── main.ts              진입점 (서버 시작)
├── test/
├── node_modules/
├── package.json
├── tsconfig.json
└── nest-cli.json
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