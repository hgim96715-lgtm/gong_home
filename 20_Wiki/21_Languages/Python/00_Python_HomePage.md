
>목표: Airflow · Spark · API 파이프라인을 직접 짤 수 있는 Python 실력

## 버전 확인 & 관리

```bash
# 현재 버전 확인
python3 --version
# 권장 버전: 3.11.x (안정적 + 속도 개선)
```

>버전 변경 방법 → [[Linux_Python_Env]] pyenv 섹션 참고

---

---

## Level 1. 기초 문법

### 변수 & 자료형

| 노트                         | 핵심 개념                                              |
| -------------------------- | -------------------------------------------------- |
| [[Python_Variables_Types]] | str / int / bool / float / type 변환 / f-string/repr |
| [[Python_Type_Checking]]   | type / isinstance / 상속 고려 여부/repr/값의 진짜 모습         |
| [[Python_Membership_In]]   | in / not in / == 완전일치                              |

### 문자열 처리

| 노트                                 | 핵심 개념                                                  |
| ---------------------------------- | ------------------------------------------------------ |
| [[Python_String_Indexing_Slicing]] | Index / Slice / [:​:-1] / 음수 인덱스                       |
| [[Python_String_Methods]]          | split / join / strip / startswith / endswith / isdigit |
| [[Python_String_Search]]           | find / index / rfind / rindex / 없으면 -1 vs 에러           |
| [[Python_String_Case_Replace]]     | upper / lower / replace / re.sub / 정규식 치환              |
| [[Python_String_Formatting]]       | f-string / :<4 / :> / :^ / 소수점 / 날짜 포맷 / 천단위           |

### 흐름 제어

|노트|핵심 개념|
|---|---|
|[[Python_Control_Flow]]|if / for / while / for-else / break / continue|
|[[Python_Looping_Helpers]]|range / enumerate / zip / unpacking|

### 함수

| 노트                           | 핵심 개념                                 |
| ---------------------------- | ------------------------------------- |
| [[Python_Functions]]         | def / return / *args / **kwargs / 기본값 |
| [[Python_Unpacking]]         | a, b = [1, 2] / *rest / _ / 중첩 언패킹    |
| [[Python_Variable_Swapping]] | a, b = b, a / 임시변수 없이 교환              |

### 내장함수 & 수학

| 노트                             | 핵심 개념                                                                  |
| ------------------------------ | ---------------------------------------------------------------------- |
| [[Python_Builtin_Functions]]   | max / min / sum / len / abs / ord / chr / pow/is_integer()/형변환/any/all |
| [[Python_Sorting_Logic]]       | sort / sorted / key / lambda / reverse / 다중 정렬                         |
| [[Python_Math_Module]]         | math.prod / ceil / floor / gcd / sqrt / pi                             |
| [[Python_Fractions_Module]]    | Fraction / 분수 연산 / 자동 약분 / 부동소수점 오차 방지                                 |
| [[Python_Statistics_Module]]   | mean / median / stdev / variance                                       |
| [[Python_Collections_Counter]] | Counter / most_common / 딕셔너리 연산                                        |

---

---

## Level 2. 자료구조

```
데이터를 담는 그릇 이해하기
JSON 처리의 핵심
```

| 노트                            | 핵심 개념                                               |
| ----------------------------- | --------------------------------------------------- |
| [[Python_Lists_Tuples]]       | [] / () / Mutable vs Immutable / append / pop,count |
| [[Python_Lambda_Map]]         | lambda / map / filter / reduce                      |
| [[Python_List_Comprehension]] | [결과 for x in 리스트 if 조건] 한 줄 축약                      |
| [[Python_Dictionaries]]       | {} / .get() / .items() / .keys() / .values()        |
| [[Python_JSON]]               | json.loads / json.dumps / indent / 직렬화·역직렬화         |
| [[Python_Sets]]               | set / 교집합 & / 합집합 \| / 차집합 - / frozenset            |

---

---

## Level 3. 모듈과 환경

```
남이 짠 코드를 가져다 쓰고, 내 코드를 정리하기
```

>[[Linux_Python_Env]] 참고 

|노트|핵심 개념|
|---|---|
|[[Python_Modules_Imports]]|import / from / **init**.py / 패키지 / 별칭 as|
|[[Python_Entry_Point]]|if **name** == '**main**' / 모듈 vs 스크립트|
|[[Python_Sys_Module]]|sys.argv / sys.path / sys.exit / 표준입출력|
|[[Python_Virtual_Env]]|venv / pip / requirements.txt / activate|
|[[Python_Regex]]|re.search / findall / sub / group / 패턴 문법|

---

---

## Level 4. 견고한 코드

```
에러가 나도 파이프라인이 멈추지 않게 하기
```

### 에러 처리

|노트|핵심 개념|
|---|---|
|[[Python_Error_Handling]]|try / except / finally / raise / Exception 종류|
|[[Python_Logging]]|logging / DEBUG·INFO·WARNING·ERROR / 핸들러|

### 파일 & 경로

|노트|핵심 개념|
|---|---|
|[[Python_File_IO]]|open / with / r·w·a 모드 / context manager|
|[[Python_OS_Module]]|os.path.exists / environ / remove / getcwd|
|[[Python_Pathlib]]|Path / exists / mkdir / / 연산자|

### API 요청 & 응답

>CS Basics [[CS_HTTP_Basics]] , [[CS_TCP_IP]], [[CS_REST_API_Methods]] ,[[Serialization_JSON_XML]] 참고 

| 노트                           | 핵심 개념                                              |
| ---------------------------- | -------------------------------------------------- |
| [[Python_Requests_Methods]]  | requests.get / post / params / headers / timeout / |
| [[Python_Requests_Response]] | status_code / .json() / raise_for_status           |
| [[HTTP_Status_Codes]]        | 200 / 404 / 500 / 429 Too Many                     |
| [[Python_URL_Parsing]]       | urlencode / quote / unquote / urlparse             |
| [[Python_ElementTree]]       | XML 파싱 / fromstring / findtext / find / findall    |

---

---

## Level 5. 객체 지향 프로그래밍 (OOP)

```
Airflow 오퍼레이터가 왜 이렇게 생겼는지 이해하기
```

|노트|핵심 개념|
|---|---|
|[[Python_Classes_Objects]]|class / self / **init** / **str** / @staticmethod / @classmethod|

---

---

## Level 6. 고급 기능

```
Airflow 2.0 과 대용량 처리의 핵심 기능
```

### 함수 고급

|노트|핵심 개념|
|---|---|
|[[Python_Decorators]]|@ 골뱅이 / @task / @wraps / TaskFlow API / 클로저|
|[[Python_Iterables_Iterators]]|iter / next / **iter** / **next**|
|[[Python_Generators_Yield]]|yield / generator / lazy evaluation / 메모리 절약|

### 날짜 & 시간

|노트|핵심 개념|
|---|---|
|[[Python_DateTime]]|datetime / timedelta / strptime / strftime / isoformat|
|[[Python_Calendar_Module]]|calendar / monthrange / isleap / weekday|

### 랜덤

|노트|핵심 개념|
|---|---|
|[[Python_Random_Seed]]|random / seed / randint / choice / uniform / 재현성|
|[[Python_Random_Sampling]]|choices / weights / k / 복원·비복원 추출|

### 기타

|노트|핵심 개념|
|---|---|
|[[Python_AST_Structure_Analysis]]|ast / literal_eval / parse / 안전한 eval|

---

---

## Level 7. 실전 시뮬레이션

```
가짜 데이터를 만들어서 파이프라인 테스트하기
```

|노트|핵심 개념|
|---|---|
|[[Python_Library_Faker]]|Faker / name / address / ip / locale|
|[[Python_Mock_Data_Generator]]|Generator / yield / 배치 생성 패턴|
|[[Python_UUID]]|uuid4 / 분산 환경 PK / 충돌 없는 식별자|

---

---

## Level 8. 외부 시스템 연결

```
파이썬이 데이터베이스 / 클라우드와 대화하기
```

|노트|핵심 개념|
|---|---|
|[[Python_Database_Connect]]|psycopg2 / connect / cursor / .env / sqlalchemy / execute / fetchall|

---

---
