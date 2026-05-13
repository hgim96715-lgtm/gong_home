---
aliases:
  - interface
  - 인터페이스
tags:
  - TypeScript
related:
  - "[[00_TS_HomePage]]"
  - "[[TS_Basic_Types]]"
  - "[[TS_Type_vs_Interface]]"
  - "[[TS_Class]]"
---

# TS_Interface — 인터페이스

## 한 줄 요약

```
객체의 모양(구조)을 정의하는 타입
"이 객체는 반드시 이런 속성을 가져야 해"
```

---

---

# ① 기본 문법

```typescript
// 인터페이스 선언
interface Movie {
  id: number;
  title: string;
}

// 사용
const movie: Movie = {
  id: 1,
  title: '마이클',
};

// 잘못된 사용
const wrong: Movie = {
  id: '1',      // ❌ string, number 필요
  title: '마이클',
};
```

---

---

# ② 선택적 속성 ?

```typescript
interface Movie {
  id: number;
  title: string;
  genre?: string;    // ? = 있어도 되고 없어도 됨
  year?: number;
}

// genre 없어도 OK
const movie: Movie = { id: 1, title: '마이클' };

// 사용 시 undefined 체크 필요
if (movie.genre) {
  console.log(movie.genre.toUpperCase());
}
```

---

---

# ③ readonly

```typescript
interface Movie {
  readonly id: number;   // 수정 불가
  title: string;
}

const movie: Movie = { id: 1, title: '마이클' };
movie.title = '마이클2';   // ✅ OK
movie.id = 2;              // ❌ 에러: readonly
```

---

---

# ④ 함수 타입 포함

```typescript
interface MovieService {
  getMovie(id: number): Movie;
  createMovie(title: string): Movie;
  deleteMovie(id: number): void;
}
```

---

---

# ⑤ interface 배열 ⭐️

```typescript
interface Movie {
  id: number;
  title: string;
}

// NestJS 에서 사용한 패턴
private movies: Movie[] = [
  { id: 1, title: '마이클' },
  { id: 2, title: '악마는 프라다를 입는다' },
];

// 함수 반환 타입
getMovies(): Movie[] {
  return this.movies;
}

getMovie(id: number): Movie {
  return this.movies.find(m => m.id === id);
}
```

---

---

# ⑥ extends — 인터페이스 확장

```typescript
interface Base {
  id: number;
  createdAt: Date;
}

interface Movie extends Base {
  title: string;
  genre: string;
}

// Movie 는 id, createdAt, title, genre 전부 가져야 함
const movie: Movie = {
  id: 1,
  createdAt: new Date(),
  title: '마이클',
  genre: '드라마',
};
```

---

---

# ⑦ type vs interface ⭐️

```typescript
// interface
interface Movie {
  id: number;
  title: string;
}

// type alias (동일한 결과)
type Movie = {
  id: number;
  title: string;
};
```

| |interface|type|
|---|---|---|
|객체 구조 정의|✅|✅|
|Union 타입|❌|✅ `string \| number`|
|extends 확장|✅ 자연스러움|✅ `&` 로 가능|
|병합 선언|✅ 같은 이름 중복 가능|❌ 에러|
|NestJS / 백엔드|주로 interface||
|프론트엔드 / 복잡한 타입||주로 type|

```
실무 관례:
  객체 모양 → interface
  Union / 복잡한 타입 → type
  NestJS DTO → interface 또는 class (class-validator 쓰면 class)
```

---

---

# NestJS 실전 패턴

```typescript
// 컨트롤러에서 interface 직접 사용
interface Movie {
  id: number;
  title: string;
}

@Controller('movie')
export class AppController {
  private movies: Movie[] = [
    { id: 1, title: '마이클' },
  ];

  @Get(':id')
  getMovie(@Param('id') id: string): Movie {
    //                              ↑ 반환 타입을 Movie 로 명시
    return this.movies.find(m => m.id === +id);
  }

  @Post()
  postMovies(@Body('title') title: string): Movie {
    const movie: Movie = {
      id: this.idCounter++,
      title,
    };
    return movie;
  }
}
```