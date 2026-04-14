---
aliases:
  - 정규표현식
  - Regex
  - REGEXP
  - 정규식
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_String_Functions]]"
  - "[[SQL_CASE_WHEN]]"
  - "[[Python_Regex]]"
---
# SQL_Regular_Expression

> **텍스트 안에서 특정 패턴을 찾고, 추출하고, 바꾸는 도구** 관련 노트 

---

## 왜 필요한가

```sql
-- LIKE 로는 "대괄호 접두어 제거" 같은 복잡한 패턴 처리 불가
WHERE title LIKE '%얼리버드%'   -- 포함 여부만 가능

-- 정규식은 패턴 자체를 기술해서 추출·변환까지 가능
REGEXP_REPLACE(title, '^\[[^\]]*\]\s*', '')  -- ［얼리버드 40%］ 제거
```

> **SQLD**: Oracle 5대장 함수 문법 + 메타문자 의미 출제 **실무**: PostgreSQL `~` 연산자 / MySQL `REGEXP` / `REGEXP_REPLACE`

---

## ⚠️ 가장 많이 헤매는 것 — 공백

```
정규식 패턴 안의 공백 = "이 위치에 반드시 공백이 있어야 한다"는 조건

매칭이 안 될 때 공백부터 의심할 것
```

```sql
-- 실제 데이터: '약 2시간 1분 후 도착예정'

-- ❌ 공백 빠뜨림 → 매칭 안 됨
'약(.+)후'

-- ✅ 공백 반영
'약 (.+) 후'

-- 공백이 불확실할 때 (있을 수도 없을 수도)
'약\s+(.+)\s+후'   -- 1개 이상
'약\s*(.+)\s*후'   -- 0개 이상
```

```sql
-- 실제 데이터 공백 위치 확인법
SELECT status, LENGTH(status) FROM 테이블 LIMIT 3;
SELECT SUBSTRING(status, 5, 1) = ' ' FROM 테이블 LIMIT 1;
```

---

## 메타문자 해독표

|기호|의미|예시|
|---|---|---|
|`.`|임의의 문자 1개|`A.C` → ABC, A1C|
|`^`|문자열 시작|`^A` → A로 시작|
|`$`|문자열 끝|`Z$` → Z로 끝|
|`[ ]`|괄호 안 문자 중 1개|`[abc]` → a 또는 b 또는 c|
|`[a-z]`|범위 지정|`[0-9]` → 숫자|
|`[^ ]`|부정 (해당 문자 제외)|`[^0-9]` → 숫자가 아닌 것|
|`( )`|그룹 (묶음 + 캡처)|`(abc)+`|
|`(?:)`|묶기만, 캡처 안 함|`(?:jpg\|png)`|
|`*`|0회 이상 반복|`ab*` → a, ab, abb|
|`+`|1회 이상 반복|`ab+` → ab, abb|
|`?`|0회 또는 1회|`apple?` → appl, apple|
|`\|`|OR|`A\|B` → A 또는 B|
|`\.`|진짜 마침표|`\.com` → .com만|
|`\(` `\)`|진짜 괄호 문자|`\(내용\)` → (내용)|
|`{n}`|정확히 n회|`[0-9]{3}` → 숫자 3자리|
|`{n,m}`|n회 이상 m회 이하|`[0-9]{2,4}`|
|`\d`|숫자 `[0-9]` 단축|PostgreSQL/MySQL|
|`\s`|공백 문자 단축|PostgreSQL/MySQL|

### POSIX 문자 클래스 — Oracle에서 주로 사용

|클래스|의미|
|---|---|
|`[:alnum:]`|영문자 + 숫자|
|`[:alpha:]`|영문자|
|`[:digit:]`|숫자 `[0-9]`|
|`[:space:]`|공백|
|`[:upper:]`|대문자|
|`[:lower:]`|소문자|

```sql
WHERE REGEXP_LIKE(컬럼, '^[[:alnum:]]+$')  -- 영문자·숫자만 포함
```

---

## ★ `^\[[^\]]*\]\s*` — 대괄호 접두어 제거 패턴

실전에서 가장 자주 쓰는 패턴 중 하나.

```
[INFO] Hello world      →  Hello world
[얼리버드 40%] 페르난도  →  페르난도
[ERROR] 시스템 오류     →  시스템 오류
```

### 패턴 분해

```
^\[[^\]]*\]\s*

^ ──────── 문자열의 맨 앞에서 시작
\[ ─────── 진짜 [ 문자  (\ 로 메타문자 해제)
[^\]]* ─── [ ] 안의 내용 (] 가 아닌 문자 0개 이상)
│          [^ ] = 부정 문자 클래스
│          \]   = ] 문자 자체 (\ 로 메타문자 해제)
│          *    = 0회 이상 (내용이 없는 [] 도 처리)
\] ─────── 진짜 ] 문자
\s* ────── 뒤따라오는 공백 0개 이상 제거
```

### 왜 `[^]]*` 인가 — `[^\]]` 핵심 설명

```
[^\]]  =  ] 가 아닌 문자 아무거나

\]     →  여기서 ] 는 진짜 ] 문자 (메타문자 아님)
[^  ]  →  이 안에서 ^ 는 부정을 의미

즉, [^\]] = "]를 제외한 모든 문자"

왜 이게 필요한가?
  [INFO] Hello]World 같은 데이터에서
  .* 를 쓰면 마지막 ] 까지 먹어버림 (탐욕적 매칭)
  [^\]]* 를 쓰면 첫 번째 ] 에서 정확히 멈춤
```

```
비교:
  \[.*\]      →  탐욕적: [INFO] Hello] 전부 먹음  ← ❌
  \[[^\]]*\]  →  첫 ] 에서 멈춤: [INFO] 만 제거  ← ✅
```

### 실제 코드

```sql
-- PostgreSQL
SELECT REGEXP_REPLACE(title, '^\[[^\]]*\]\s*', '')
FROM exhibitions;
-- '［얼리버드 40%］ 페르난도 보테로展'  →  '페르난도 보테로展'
-- '[INFO] 시스템 오류'                  →  '시스템 오류'
-- '정상 제목'                           →  '정상 제목' (변화 없음)

-- MySQL
SELECT REGEXP_REPLACE(title, '^\\[[^\\]]*\\]\\s*', '')
FROM exhibitions;
-- MySQL에서는 \ 를 \\ 로 이중 이스케이프

-- Oracle
SELECT REGEXP_REPLACE(title, '^\[[^\]]*\]\s*', '')
FROM exhibitions;
-- Oracle은 PostgreSQL과 동일 문법

-- dbt (stg_exhibitions.sql 실제 사용)
TRIM(REGEXP_REPLACE(title, '^\[[^\]]*\]\s*', '')) AS title_cleaned
```

### 변형 패턴

```sql
-- 중간에 있는 대괄호도 모두 제거 (접두어 아닌 것까지)
REGEXP_REPLACE(title, '\[[^\]]*\]', '')   -- ^ 없애면 전체 탐색

-- 맨 앞 소괄호 접두어도 제거: (INFO) Hello → Hello
REGEXP_REPLACE(title, '^\([^\)]*\)\s*', '')

-- 맨 앞 숫자 접두어 제거: 01. 제목 → 제목
REGEXP_REPLACE(title, '^\d+\.\s*', '')
```

---

## DB별 정규식 함수 비교

|목적|Oracle (SQLD)|PostgreSQL (실무)|MySQL|
|---|---|---|---|
|패턴 일치 여부|`REGEXP_LIKE(컬럼, '패턴')`|`컬럼 ~ '패턴'`|`컬럼 REGEXP '패턴'`|
|대소문자 무시|`REGEXP_LIKE(컬럼, '패턴', 'i')`|`컬럼 ~* '패턴'`|`컬럼 REGEXP '(?i)패턴'`|
|불일치|—|`컬럼 !~ '패턴'`|`컬럼 NOT REGEXP '패턴'`|
|패턴 추출|`REGEXP_SUBSTR(컬럼, '패턴')`|`SUBSTRING(컬럼 FROM '패턴')`|`REGEXP_SUBSTR(컬럼, '패턴')` ⭐ 8.0+|
|그룹 추출|`REGEXP_SUBSTR(..., 그룹번호)`|`(REGEXP_MATCH(컬럼, '패턴'))[1]`|`REGEXP_SUBSTR(컬럼, '패턴', 1, 1, '', 그룹번호)`|
|치환|`REGEXP_REPLACE(컬럼, '패턴', '교체')`|`REGEXP_REPLACE(컬럼, '패턴', '교체')`|`REGEXP_REPLACE(컬럼, '패턴', '교체')` ⭐ 8.0+|
|개수 세기|`REGEXP_COUNT(컬럼, '패턴')`|전용 함수 없음 (우회)|전용 함수 없음 (우회)|
|위치 찾기|`REGEXP_INSTR(컬럼, '패턴')`|전용 함수 없음|전용 함수 없음|

---

## Oracle — 5대장 함수 (SQLD 핵심)

### 매개변수 공통 순서

```
REGEXP_함수(대상, '패턴', [시작위치], [발생횟수], ['옵션'], [그룹번호])
                           기본:1       기본:1      기본:'c'   기본:0
```

|옵션|의미|
|---|---|
|`'i'`|대소문자 무시|
|`'c'`|대소문자 구분 (기본)|
|`'m'`|다중행 모드|

### REGEXP_LIKE — 조건 판별 (WHERE에서만)

```sql
-- TRUE/FALSE 반환 → WHERE / HAVING 에서만 사용
WHERE REGEXP_LIKE(단어, '^A..D$')
WHERE REGEXP_LIKE(이름, '^a', 'i')   -- 대소문자 무시
```

### REGEXP_SUBSTR — 추출

```sql
-- 주소에서 우편번호 추출
SELECT REGEXP_SUBSTR(주소, '[0-9]{3}') FROM 고객정보;

-- 이메일에서 @ 뒤 도메인 (2번째 덩어리)
SELECT REGEXP_SUBSTR(이메일, '[^@]+', 1, 2) FROM 사원;
--                                     ↑  ↑
--                                     1번째 글자부터, 2번째 일치
```

### REGEXP_REPLACE — 치환 (Oracle·PG·MySQL 공통)

```sql
-- 주민번호 뒷자리 마스킹
SELECT REGEXP_REPLACE(주민번호, '-[0-9]{7}', '-*******') FROM 직원;

-- 전화번호 하이픈 제거
SELECT REGEXP_REPLACE('010-1234-5678', '-', '') FROM DUAL;
-- 결과: 01012345678
```

### REGEXP_INSTR — 위치 반환 (Oracle 전용, SQLD 출제)

```sql
SELECT REGEXP_INSTR('데이터123엔지니어', '[0-9]+') FROM DUAL;
-- 데(1) 이(2) 터(3) 1(4) → 결과: 4

-- 발생 횟수 활용 (SQLD 기출)
SELECT REGEXP_INSTR('abc123def456', '[a-z]+', 1, 2) FROM DUAL;
-- 2번째 소문자 덩어리 'def' 의 시작 위치 → 결과: 7
```

### REGEXP_COUNT — 개수 (Oracle 전용, SQLD 출제)

```sql
SELECT REGEXP_COUNT('A123-B45-C6789', '\d+') FROM DUAL;
-- '123'(1) '45'(2) '6789'(3) → 결과: 3
```

---

## PostgreSQL — 실무 사용법

### `~` 연산자 — 패턴 일치 여부

```sql
WHERE status ~ '^운행 중'      -- 시작 일치
WHERE 이름 ~* '^kim'           -- 대소문자 무시
WHERE 상태 !~ '완료'           -- 불일치
```

### `SUBSTRING FROM` — 패턴으로 추출

```sql
-- ( ) 그룹 있으면 그룹 내용만 반환
SELECT SUBSTRING(status FROM '약 (.+) 후 도착예정')
FROM train_realtime;
-- '약 2시간 1분 후 도착예정' → '2시간 1분'
```

### `REGEXP_MATCH` — 그룹 배열 반환

```sql
-- 단일 그룹
SELECT (REGEXP_MATCH(status, '약 (.+) 후 도착예정'))[1] FROM 테이블;

-- 복수 그룹
SELECT
    (REGEXP_MATCH('010-1234-5678', '(\d{3})-(\d{4})-(\d{4})'))[1],  -- '010'
    (REGEXP_MATCH('010-1234-5678', '(\d{3})-(\d{4})-(\d{4})'))[2],  -- '1234'
    (REGEXP_MATCH('010-1234-5678', '(\d{3})-(\d{4})-(\d{4})'))[3];  -- '5678'

-- 매칭 실패 시 기본값
SELECT COALESCE((REGEXP_MATCH(status, '약 (.+) 후'))[1], '-') FROM 테이블;
```

### `REGEXP_REPLACE` + 역참조 `\1`

```sql
-- 전화번호 형식 변환: 01012345678 → 010-1234-5678
SELECT REGEXP_REPLACE('01012345678', '(\d{3})(\d{4})(\d{4})', '\1-\2-\3');

-- 이름 순서 바꾸기: '홍 길동' → '길동 홍'
SELECT REGEXP_REPLACE('홍 길동', '(\S+) (\S+)', '\2 \1');
```

### `REGEXP_SPLIT_TO_TABLE` — 패턴으로 행 분리

```sql
SELECT REGEXP_SPLIT_TO_TABLE('서울,부산,대전', ',');
-- 서울 / 부산 / 대전 (각각 행으로)
```

---

## MySQL — 실무 사용법 (8.0+)

### `REGEXP` / `RLIKE` — 패턴 일치 여부

```sql
-- REGEXP 와 RLIKE 는 동일 (별칭 관계)
WHERE title REGEXP '^\\[.*\\]'     -- [ 로 시작하는 행
WHERE status REGEXP '운행'
WHERE status NOT REGEXP '완료'

-- MySQL은 기본이 대소문자 무시 (BINARY 키워드로 구분 가능)
WHERE title REGEXP BINARY '^A'     -- 대소문자 구분
```

> MySQL `REGEXP`는 `REGEXP_LIKE()` 함수와 동일

### `REGEXP_SUBSTR` — 추출 (MySQL 8.0+)

```sql
-- 기본 추출
SELECT REGEXP_SUBSTR('서울 강남구 역삼동', '[가-힣]+구');
-- 결과: 강남구

-- 그룹 추출 (6번째 인자: 그룹 번호)
SELECT REGEXP_SUBSTR('010-1234-5678', '(\\d{3})-(\\d{4})', 1, 1, '', 1);
-- 결과: 010
```

### `REGEXP_REPLACE` — 치환 (MySQL 8.0+)

```sql
-- 대괄호 접두어 제거 (MySQL은 \ 를 \\ 로 이중 이스케이프)
SELECT REGEXP_REPLACE(title, '^\\[[^\\]]*\\]\\s*', '')
FROM exhibitions;
-- '[INFO] Hello world'  →  'Hello world'

-- 전화번호 형식 통일
SELECT REGEXP_REPLACE('010 1234 5678', '\\s', '-');
-- 결과: 010-1234-5678

-- 역참조 사용 (MySQL도 \1 지원)
SELECT REGEXP_REPLACE('01012345678', '(\\d{3})(\\d{4})(\\d{4})', '\\1-\\2-\\3');
-- 결과: 010-1234-5678
```

### MySQL 이스케이프 주의

```
PostgreSQL / Oracle:  \[  →  진짜 [ 문자
MySQL:               \\[ →  진짜 [ 문자  (\ 하나를 더 써야 함)

이유: MySQL은 문자열 파싱 → 정규식 파싱 순서로 두 번 처리
  '\\['  →  문자열 파싱 후 '\['  →  정규식에서 진짜 [

실수 패턴:
  MySQL에서 \d 쓰면 안 됨 → \\d 또는 [0-9] 사용
  MySQL에서 \[ 쓰면 안 됨 → \\[ 사용
```

---

## SQLD 킬러 함정 2가지

### ① 중첩(Overlapping) 불가 — 한 번 소비한 문자는 재사용 안 함

```sql
SELECT REGEXP_COUNT('123123123', '123123') FROM DUAL;
-- 사람의 생각: 2  ← ❌
-- 컴퓨터 처리: '123123' 소비 → 남은 '123' 은 패턴(6자리) 불일치
-- 결과: 1        ← ✅
```

### ② 중첩 그룹 번호 — 왼쪽 `(` 순서대로

```sql
REGEXP_SUBSTR('1234567890', '(123)(4(56)(78))', 1, 1, 'i', 그룹번호)
```

```
( 1 2 3 )( 4 ( 5 6 )( 7 8 ))
  Group1    Group2  G3   G4

그룹번호 0 → 12345678  (전체 매칭)
그룹번호 1 → 123       (첫 번째 그룹)
그룹번호 2 → 45678     (두 번째 그룹 전체 — 안쪽 포함)
그룹번호 3 → 56
그룹번호 4 → 78
```

---

## `( )` vs `(?:)` — 캡처 vs 비캡처

```
( )   → 묶기 + 캡처  : \1, \2 로 꺼낼 수 있음
(?:)  → 묶기만       : 번호 안 매겨짐, 꺼낼 수 없음
```

```sql
-- ❌ 불필요한 그룹이 \1 을 차지해서 번호 밀림
'(운행 중) \([0-9]+%진행, 약 (.+) 후\)'
-- \1 = '운행 중'  ← 원하지 않음
-- \2 = '2시간 1분'

-- ✅ 필요없는 그룹은 (?:) 로
'(?:운행 중) \([0-9]+%진행, 약 (.+) 후\)'
-- \1 = '2시간 1분'  ← 번호 안 밀림
```

```sql
-- optional 패턴 — 있을 수도 없을 수도
-- "2시간 30분" 또는 "30분" 둘 다 매칭
WHERE status ~ '(?:[0-9]+시간 )?[0-9]+분'

-- dbt stg 실제 사용
SELECT (REGEXP_MATCH(status, '약 ((?:[0-9]+시간 )?[0-9]+분) 후'))[1]
-- '약 2시간 1분 후' → '2시간 1분'
-- '약 45분 후'      → '45분'
```

---

## Oracle vs PostgreSQL vs MySQL 변환 대조표

```sql
-- ① 패턴 포함 여부
-- Oracle
WHERE REGEXP_LIKE(status, '^운행')
-- PostgreSQL
WHERE status ~ '^운행'
-- MySQL
WHERE status REGEXP '^운행'

-- ② 그룹 추출
-- Oracle
SELECT REGEXP_SUBSTR(status, '약 (.+) 후', 1, 1, 'i', 1) FROM ...
-- PostgreSQL
SELECT (REGEXP_MATCH(status, '약 (.+) 후'))[1] FROM ...
-- MySQL (8.0+)
SELECT REGEXP_SUBSTR(status, '약 (.+) 후', 1, 1, '', 1) FROM ...

-- ③ 치환 (셋 다 문법 유사, MySQL만 이스케이프 주의)
-- Oracle / PostgreSQL
SELECT REGEXP_REPLACE(전화번호, '(\d{3})(\d{4})(\d{4})', '\1-\2-\3')
-- MySQL
SELECT REGEXP_REPLACE(전화번호, '(\\d{3})(\\d{4})(\\d{4})', '\\1-\\2-\\3')

-- ④ 대괄호 접두어 제거
-- Oracle / PostgreSQL
REGEXP_REPLACE(title, '^\[[^\]]*\]\s*', '')
-- MySQL
REGEXP_REPLACE(title, '^\\[[^\\]]*\\]\\s*', '')
```

---

## 자주 쓰는 패턴 모음

```sql
-- 숫자만 추출
REGEXP_REPLACE(컬럼, '[^0-9]', '')            -- 숫자 아닌 것 전부 제거

-- 전화번호 형식 통일 (하이픈 제거)
REGEXP_REPLACE(전화번호, '[-\s]', '')

-- 이메일 도메인 추출
SUBSTRING(이메일 FROM '@(.+)$')               -- PostgreSQL

-- 날짜 패턴 감지
컬럼 ~ '\d{4}-\d{2}-\d{2}'                   -- PostgreSQL

-- 가격 패턴 감지 (1,000 형태)
컬럼 ~ '\d{1,3},\d{3}'                       -- PostgreSQL

-- 맨 앞 특수문자 제거: [태그] 제목 → 제목
REGEXP_REPLACE(title, '^\[[^\]]*\]\s*', '')  -- ★ 핵심 패턴

-- 연속 공백 단일화
REGEXP_REPLACE(텍스트, '\s+', ' ', 'g')      -- PostgreSQL (g = 전체 치환)
```