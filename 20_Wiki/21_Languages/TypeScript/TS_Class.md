---
aliases:
  - TypeScript 클래스
  - class
  - 접근 제어자
tags:
  - TypeScript
related:
  - "[[00_TS_HomePage]]"
  - "[[TS_Interface]]"
  - "[[JS_Class]]"
  - "[[NestJS_Controller]]"
---

# TS_Class — 클래스

## 한 줄 요약

```
JS 클래스 + TypeScript 접근 제어자 (public/private/readonly)
NestJS 의 핵심 = 클래스 기반
```

---

---

# ① 기본 클래스 구조

```typescript
class Person {
  name: string;
  age: number;

  constructor(name: string, age: number) {
    this.name = name;
    this.age = age;
  }

  greet(): string {
    return `Hi, I'm ${this.name}`;
  }
}

const person = new Person('홍길동', 30);
person.greet();  // 'Hi, I'm 홍길동'
```

---

---

# ② 접근 제어자 ⭐️

```typescript
class Movie {
  public id: number;       // 어디서든 접근 가능 (기본값)
  private title: string;   // 클래스 내부에서만
  protected year: number;  // 클래스 + 자식 클래스에서
  readonly genre: string;  // 읽기만 / 수정 불가

  constructor(id: number, title: string, year: number, genre: string) {
    this.id = id;
    this.title = title;
    this.year = year;
    this.genre = genre;
  }
}

const movie = new Movie(1, '마이클', 2023, '드라마');
movie.id = 2;       // ✅ public
movie.title = '수정'; // ❌ private
movie.genre = '액션'; // ❌ readonly
```

|제어자|클래스 내부|자식 클래스|외부|
|---|:-:|:-:|:-:|
|`public`|✅|✅|✅|
|`protected`|✅|✅|❌|
|`private`|✅|❌|❌|
|`readonly`|✅ (생성자만 수정)|✅|읽기만|

---

---

# ③ constructor 단축 문법 ⭐️

```typescript
// 기존 방식 — 속성 선언 + 할당 2번
class Movie {
  id: number;
  title: string;

  constructor(id: number, title: string) {
    this.id = id;
    this.title = title;
  }
}

// 단축 방식 — constructor 매개변수에 접근 제어자 붙이기
class Movie {
  constructor(
    public id: number,
    private title: string,
  ) {}
  // 자동으로 this.id = id / this.title = title 처리됨
}
```

```
NestJS 에서 자주 보이는 패턴:
constructor(private readonly appService: AppService) { }
            ↑       ↑        ↑
          private  readonly  타입(DI 주입)

→ 클래스 속성 자동 생성 + 읽기 전용 설정
→ 외부에서 appService 변경 불가
```

---

---

# ④ NestJS 컨트롤러 클래스 분석

```typescript
@Controller('movie')
export class AppController {

  // 클래스 속성 선언
  private movies: Movie[] = [
    { id: 1, title: '마이클' },
  ];
  private idCounter: number = 3;

  // constructor — 의존성 주입
  constructor(private readonly appService: AppService) { }
  //          ↑ 단축 문법으로 클래스 속성 자동 생성
  //                  ↑ 수정 불가

  // 메서드 (클래스 함수)
  @Get()
  getMovies(): Movie[] {
    return this.movies;
    //     ↑ this = 클래스 인스턴스
  }
}
```

---

---

# ⑤ extends — 상속

```typescript
class Animal {
  constructor(public name: string) {}

  speak(): string {
    return `${this.name} makes a sound`;
  }
}

class Dog extends Animal {
  constructor(name: string, public breed: string) {
    super(name);   // 부모 constructor 호출 (필수)
  }

  speak(): string {
    return `${this.name} barks!`;   // 오버라이드
  }
}

const dog = new Dog('Rex', '진돗개');
dog.speak();   // 'Rex barks!'
```

---

---

# ⑥ abstract — 추상 클래스

```typescript
abstract class BaseService {
  // 구현 없이 선언만 (자식 클래스에서 반드시 구현)
  abstract getData(): any[];

  // 공통 메서드는 구현 가능
  log(msg: string): void {
    console.log(`[LOG] ${msg}`);
  }
}

class MovieService extends BaseService {
  getData(): Movie[] {
    return [];   // 반드시 구현
  }
}

// abstract 클래스는 직접 인스턴스 생성 불가
new BaseService();    // ❌ 에러
new MovieService();   // ✅
```

---

---

# 접근 제어자 선택 가이드

```
외부에서 써야 함     → public (또는 생략)
내부에서만 써야 함   → private
상속해서 쓸 것       → protected
절대 바뀌면 안 됨    → readonly
의존성 주입 (NestJS) → private readonly
```