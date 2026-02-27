## Level 1. 기초 문법 (Syntax)

> **"파이썬과 대화하는 법을 배워요."**

- [[Python_Variables_Types]] : 변수와 자료형 (`str`, `int`, `bool`, `float`, `type 변환`, `f-string`)
- **[[Python_Membership_In]]** : 있다? 없다? 멤버십 연산자 (`in`, `not in`, `{text}==`, `완전일치`)
- [[Python_String_Indexing_Slicing]] : 문자열 조각내기 (`Index`, `Slice`, `[:​:-1]`, `음수 인덱스`)
- [[Python_Control_Flow]] : 흐름 제어하기 (`if`, `for`, `while`, `for-else`, `break`, `continue`)
- [[Python_Looping_Helpers]] : 반복문의 3대장 (`range`, `enumerate`, `zip`, `unpacking`)
- [[Python_String_Methods]] : 문자열 가지고 놀기 (`split`, `join`, `strip`, `startswith`, `endswith`, `isdigit`, `isalpha`, `lower`, `translate`, `maketrans`)
- [[Python_String_Search]] : 문자열 검색, 위치 찾기 (`find`, `index`, `rfind`, `rindex`, `없으면 -1 vs 에러`)
- [[Python_String_Case_Replace]] : 대소문자 변환과 치환 (`upper`, `lower`, `replace`, `re.sub`, `정규식 치환`)
- [[Python_Sorting_Logic]] : 정렬의 미학 (`sort`, `sorted`, `key`, `lambda`, `reverse`, `다중 정렬`)
- [[Python_Builtin_Functions]] : 코드를 줄여주는 마법 (`max`, `min`, `sum`, `len`, `abs`, `ord`, `chr`, `pow`, `bit_length`)
- **[[Python_Type_Checking]]** : 확실하게 자료형 검사하기 (`type`, `isinstance`, `상속 고려 여부`)
- [[Python_Math_Module]] : 정밀한 수학 연산 (`math.prod`, `ceil`, `floor`, `gcd`, `sqrt`, `pi`)
- [[Python_Statistics_Module]] : 데이터 요약 통계 (`mean`, `median`, `stdev`, `variance`)
- [[Python_Collections_Counter]] : 빈도 분석의 강자 (`Counter`, `most_common`, `딕셔너리 연산`)
- [[Python_Functions]] : 함수 정의와 인자 (`def`, `return`, `*args`, `**kwargs`, `기본값`)
- **[[Python_Unpacking]]** : 리스트/튜플 한 방에 풀기 (`a, b = [1, 2]`, `*rest`, `_`, `중첩 언패킹`)
- [[Python_Variable_Swapping]] : 파이썬 변수 교환 (`a, b = b, a`, `임시변수 없이 교환`)

---

## Level 2. 데이터 자료구조 (Data Structures)

> **"데이터를 담는 그릇을 이해해요. (JSON 처리의 핵심)"**

- [[Python_Lists_Tuples]] : 순서가 있는 데이터 (`[]`, `()`, `Mutable vs Immutable`, `append`, `pop`)
- [[Python_List_Comprehension]] : 리스트 컴프리헨션 심화 (`[x for x in]`, `조건 필터`, `중첩 반복`)
- [[Python_Dictionaries]] : 키-값 쌍 데이터 (`{}`, `.get()`, `.items()`, `.keys()`, `.values()`, `dict`, `fromkeys`)
- [[Python_JSON]] : 딕셔너리와 문자열 변환 (`json.loads`, `json.dumps`, `indent`, `직렬화/역직렬화`)
- [[Python_Sets]] : 중복 제거하기 (`set`, `교집합 &`, `합집합 |`, `차집합 -`, `frozenset`)

---

## Level 3. 모듈과 환경 (Modularization)

> **"남이 짠 코드를 가져다 쓰고, 내 코드를 정리해요."**

- [[Python_Modules_Imports]] : `import`와 `from`의 차이 (`__init__.py`, `패키지`, `별칭 as`)
- [[Python_Sys_Module]] : 시스템 제어 (`sys.argv`, `sys.path`, `sys.exit`, `표준입출력`)
- [[Python_Entry_Point]] : 프로그램 시작점 (`if __name__ == '__main__':`, `모듈 vs 스크립트`)
- [[Python_Regex]] : 정규표현식 마스터 (`re.search`, `findall`, `sub`, `group`, `패턴 문법`)
- [[Python_Virtual_Env]] : 가상 환경 관리 (`venv`, `pip`, `requirements.txt`, `activate`)

---

## Level 4. 견고한 코드 만들기 (Robustness)

> **"에러가 나도 파이프라인이 멈추지 않게 해요."**

- [[Python_Error_Handling]] : 예외 처리 (`try`, `except`, `finally`, `raise`, `Exception 종류`)
- **[[Python_OS_Module]]** : 운영체제 제어의 원조 (`os.path.exists`, `environ`, `remove`, `getcwd`, `listdir`)
- [[Python_File_IO]] : 파일 읽고 쓰기 (`open`, `with`, `r/w/a 모드`, `context manager`)
- [[Python_Pathlib]] : `os.path`보다 똑똑한 현대적 경로 처리 (`exists`, `mkdir`, `Path`, `/` 연산자)
- [[Python_Logging]] : `print`는 그만! 로그 남기기 (`logging`, `DEBUG/INFO/WARNING/ERROR`, `핸들러`)
- [[Python_Requests_Methods]] : API 요청 보내기 (`requests.get`, `post`, `params`, `data`, `headers`, `timeout`)
- [[Python_Requests_Response]] : 응답 데이터 처리 (`status_code`, `.json()`, `text`, `raise_for_status`)
- [[HTTP_Status_Codes]] : 응답 코드 해석 (`200 OK`, `404 Not Found`, `500 Server Error`, `429 Too Many`)

---

## Level 5. 객체 지향 프로그래밍 (OOP)

> **"Airflow 오퍼레이터가 왜 이렇게 생겼는지 이해해요."**

- [[Python_Classes_Objects]] : 클래스와 인스턴스 (`class`, `self`, `__init__`, `__str__`, `__repr__`)
- [[Python_Inheritance]] : 상속과 오버라이딩 (`super()`, `BaseOperator 상속`, `다형성`, `메서드 오버라이딩`)

---

## Level 6. 고급 기능 (Advanced for DE)

> **"Airflow 2.0과 대용량 처리의 핵심 기능을 써봐요."**

- [[Python_Decorators]] : 골뱅이(`@`)의 정체 (`@task`, `@wraps`, `TaskFlow API`, `클로저`)
- [[Python_Iterables_Iterators]] : 반복 가능한 객체의 원리 (`iter`, `next`, `__iter__`, `__next__`)
- [[Python_Generators_Yield]] : 메모리를 아끼는 반복문 (`yield`, `generator`, `lazy evaluation`)
- [[Python_Lambda_Map]] : 익명 함수와 고속 처리 (`lambda`, `map`, `filter`, `reduce`)
- [[Python_DateTime]] : 날짜 계산과 스케줄링 (`datetime`, `timedelta`, `strptime`, `strftime`, `isoformat`, `weekday`)
- [[Python_Random_Seed]] : 랜덤 시드 고정 (`random`, `seed`, `randint`, `choice`, `uniform`, `재현성`)
- [[Python_Random_Sampling]] : 랜덤 추출과 가중치 (`choices`, `weights`, `k`, `복원/비복원 추출`)
- [[Python_AST_Structure_Analysis]] : 코드 구조 분석 (`ast`, `literal_eval`, `parse`, `안전한 eval`)
- **[[Python_Calendar_Module]]** : 달력 규칙과 윤년 계산의 마법사 (`calendar`, `monthrange`, `isleap`, `weekday`)

---

## Level 7. 실전 시뮬레이션 (Simulation & Utils)

> **"직접 가짜 데이터를 만들어서 파이프라인을 테스트해요."**

- [[Python_Library_Faker]] : 그럴싸한 가짜 데이터 생성 (`Faker`, `name`, `address`, `ip`, `locale`)
- [[Python_Mock_Data_Generator]] : 직접 만드는 데이터 생성기 패턴 (`Generator`, `yield`, `배치 생성`)
- **[[Python_UUID]]** : 유일한 ID 만들기 (`uuid4`, `분산 환경 PK`, `충돌 없는 식별자`)

---

## Level 8. 외부 시스템 연결 (Integration)

> **"파이썬이 데이터베이스나 클라우드와 대화해요."**

- **[[Python_Database_Connect]]** : 파이썬으로 SQL 실행하기 (`psycopg2`, `connect`, `cursor`, `.env`, `sqlalchemy`, `create_engine`, `execute`, `fetchall`)