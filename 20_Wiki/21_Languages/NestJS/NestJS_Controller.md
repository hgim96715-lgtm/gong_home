---
aliases:
  - NestJS 컨트롤러
  - Controller
  - 라우팅
tags:
  - NestJS
related:
  - "[[00_NestJS_HomePage]]"
  - "[[NestJS_LifeCycle]]"
  - "[[NestJS_Service_Provider]]"
  - "[[NestJS_Pipe]]"
  - "[[TS_Interface]]"
---

# NestJS_Controller — 컨트롤러

## 한 줄 요약

```
URL 경로와 HTTP 메서드를 연결하는 라우터
요청을 받아서 Service 로 넘기고 응답을 반환
비즈니스 로직은 직접 처리하지 않음
```

---

---

# ① @Controller — 기본 구조

```typescript
import { Controller, Get, Post, Patch, Delete } from '@nestjs/common';

@Controller('movie')   // 기본 경로 = /movie
export class AppController {

  @Get()               // GET /movie
  getMovies() { }

  @Post()              // POST /movie
  postMovie() { }

  @Patch(':id')        // PATCH /movie/:id
  patchMovie() { }

  @Delete(':id')       // DELETE /movie/:id
  deleteMovie() { }
}
```

```
@Controller('movie'):
  이 컨트롤러의 모든 경로 앞에 /movie 를 붙임
  @Controller() (비어있으면) → 루트 경로 /

@Get() / @Post() 등:
  HTTP 메서드 지정
  괄호 안에 추가 경로 지정 가능
    @Get()       → GET /movie
    @Get(':id')  → GET /movie/:id
```

## REST API 경로 패턴 ⭐️

```
GET    /movie          목록 조회
GET    /movie/:id      단건 조회
POST   /movie          생성
PATCH  /movie/:id      수정
DELETE /movie/:id      삭제

:id = Path Variable (경로 변수)
  /movie/1 → id = "1"
  /movie/2 → id = "2"
```

---

---

# ② @Param — Path Variable ⭐️

```typescript
import { Controller, Get, Param, NotFoundException } from '@nestjs/common';

@Get(':id')
getMovie(@Param('id') id: string): any {
  const movie = this.movies.find(movie => movie.id === parseInt(id));

  if (!movie) {
    throw new NotFoundException(`${id} 이라는 id의 영화는 존재하지 않습니다.`);
  }

  return movie;
}
```

```
@Param('id') id: string
  ↑           ↑    ↑
  데코레이터   변수명 타입

@Get(':id') 에서 :id 로 선언한 부분을
@Param('id') 로 꺼내서 사용

타입 주의:
  URL 에서 받은 값은 항상 string
  숫자로 비교하려면 변환 필요

  parseInt(id)    → 정수 변환
  +id             → 숫자 변환 (단축형)
  Number(id)      → 숫자 변환

  movie.id === parseInt(id)  ✅
  movie.id === id            ❌ (string vs number 비교)
```

## Postman 에서 Path Variable 사용

```
GET http://localhost:3000/movie/1
                                ↑
                              :id = 1

Postman → Params 탭 → Path Variables
  KEY   = id
  VALUE = 1
```

---

---

# ③ @Body — 요청 Body 데이터 ⭐️

```typescript
import { Controller, Post, Body } from '@nestjs/common';

@Post()
postMovies(@Body('title') title: string): any {
  const movie: Movie = {
    id: this.idCounter++,
    title,     // title: title 단축형
  };
  this.movies.push(movie);
  return movie;
}
```

```
@Body('title') title: string
  → 요청 Body 의 title 필드만 꺼내기

@Body() body: any
  → Body 전체를 객체로 받기

JSON Body 형식 (Postman):
  {
    "title": "겨울왕국"
  }

⚠️ Key 에 반드시 큰따옴표("") 사용:
  { "title": "겨울왕국" }  ✅
  { 'title': '겨울왕국' }  ❌ JSON 파싱 에러
```

## PATCH 에서 @Param + @Body 함께

```typescript
@Patch(':id')
patchMovies(
  @Param('id') id: string,
  @Body('title') title: string
): any {
  const movie = this.movies.find(movie => movie.id === +id);

  if (!movie) {
    throw new NotFoundException(`${id} 이라는 id의 영화는 존재하지 않습니다.`);
  }

  movie.title = title;
  return movie;
}
```

```
PATCH 는 어떤 항목을 수정할지 알아야 함
→ :id 로 대상 지정 (Path Variable)
→ Body 로 수정할 데이터 전달

POST 는 새로 만드는 거라 :id 불필요
DELETE 는 Body 데이터 불필요
```

---

---

# ④ @Query — Query Parameter ⭐️

```typescript
import { Controller, Get, Query } from '@nestjs/common';

@Get()
getMovies(@Query('title') title: string): any {
  if (!title) {
    return this.movies;   // title 없으면 전체 반환
  }

  // 정확히 일치
  return this.movies.filter(movie => movie.title === title);

  // 또는 앞부분 일치 (startsWith)
  return this.movies.filter(movie => movie.title.startsWith(title));
}
```

```
Query Parameter:
  URL 뒤에 ?key=value 형태로 전달
  GET /movie?title=마이클

  @Query('title') → ?title= 값 꺼내기
  값이 없으면 undefined → !title 로 체크

Path Variable vs Query Parameter:
  /movie/:id     → 특정 리소스 식별 (필수)
  /movie?title=  → 필터링 / 검색 (선택)
```

## Postman 에서 Query Parameter

```
GET http://localhost:3000/movie?title=마이클

Postman → Params 탭 → Query Params
  KEY   = title
  VALUE = 마이클
```

---

---

# ⑤ NotFoundException — HTTP 에러 반환 ⭐️

```typescript
import { NotFoundException } from '@nestjs/common';

// 일반 Error → 500 Internal Server Error
throw new Error('존재하지 않는 ID 입니다.');

// NotFoundException → 404 Not Found (자동으로 형식 맞춤)
throw new NotFoundException(`${id} 이라는 id의 영화는 존재하지 않습니다.`);
```

```
NestJS 응답 형식 자동 생성:
{
  "message": "3 이라는 id의 영화는 존재하지 않습니다.",
  "error": "Not Found",
  "statusCode": 404
}

자주 쓰는 예외 클래스:
  NotFoundException      404 Not Found
  BadRequestException    400 Bad Request
  UnauthorizedException  401 Unauthorized
  ForbiddenException     403 Forbidden
  ConflictException      409 Conflict

```

→ [[NestJS_Exception]] 참고

---

---

# ⑥ 실전 CRUD 컨트롤러 전체 코드

```typescript
import {
  Controller, Get, Post, Patch, Delete,
  Param, Body, Query, NotFoundException
} from '@nestjs/common';

interface Movie {
  id: number;
  title: string;
}

@Controller('movie')
export class AppController {

  private movies: Movie[] = [
    { id: 1, title: '마이클' },
    { id: 2, title: '악마는 프라다를 입는다' },
  ];
  private idCounter: number = 3;

  // GET /movie  /  GET /movie?title=검색어
  @Get()
  getMovies(@Query('title') title: string): Movie[] {
    if (!title) return this.movies;
    return this.movies.filter(m => m.title.startsWith(title));
  }

  // GET /movie/:id
  @Get(':id')
  getMovie(@Param('id') id: string):  Movie {
    const movie = this.movies.find(m => m.id === +id);
    if (!movie) throw new NotFoundException(`${id} 영화 없음`);
    return movie;
  }

  // POST /movie
  @Post()
  postMovies(@Body('title') title: string):  Movie {
    const movie: Movie = { id: this.idCounter++, title };
    this.movies.push(movie);
    return movie;
  }

  // PATCH /movie/:id
  @Patch(':id')
  patchMovies(@Param('id') id: string, @Body('title') title: string): Movie{
    const movie = this.movies.find(m => m.id === +id);
    if (!movie) throw new NotFoundException(`${id} 영화 없음`);
    movie.title = title;
    return movie;
  }

  // DELETE /movie/:id
  @Delete(':id')
  deleteMovies(@Param('id') id: string){
    const index = this.movies.findIndex(m => m.id === +id);
    if (index === -1) throw new NotFoundException(`${id} 영화 없음`);
    this.movies.splice(index, 1);
    return id;
  }
}
```

---

---

# 데코레이터 한눈에

|데코레이터|역할|예시|
|---|---|---|
|`@Controller('경로')`|기본 경로 설정|`@Controller('movie')`|
|`@Get()` / `@Get(':id')`|GET 라우팅|`@Get(':id')`|
|`@Post()`|POST 라우팅||
|`@Patch(':id')`|PATCH 라우팅||
|`@Delete(':id')`|DELETE 라우팅||
|`@Param('id')`|Path Variable 꺼내기|`/movie/1` → id=1|
|`@Body('title')`|Body 데이터 꺼내기|`{"title":"값"}`|
|`@Query('title')`|Query Parameter|`?title=값`|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`movie.id === id` 비교 오류|string vs number|`+id` 또는 `parseInt(id)`|
|JSON Body 파싱 에러|작은따옴표 사용|`"key"` 반드시 큰따옴표|
|404 대신 500 반환|`new Error()` 사용|`new NotFoundException()`|
|PATCH 에서 전체 데이터 날아감|할당 방식|`movie.title = title` (전체 교체 말고 필드만)|