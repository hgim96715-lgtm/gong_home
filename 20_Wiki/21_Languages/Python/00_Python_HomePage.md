
## Level 1. 기초 문법 (Syntax)

> "파이썬과 대화하는 법을 배워요."

- [[Python_Variables_Types]] : 변수와 자료형 (`string`,`int`,`bool`)
- [[Python_String_Indexing_Slicing]] : 문자열 조각내기 (Index, Slice,`[`:`:-1]`)
- [[Python_Control_Flow]] : 흐름 제어하기 (`if`,`for`,`while`)
- [[Python_Looping_Helpers]] : 반복문의 3대장 (`range`, `enumerate`, `zip`)
- [[Python_String_Methods]] : 문자열 가지고 놀기 (`split`, `endswith`, `join`, `strip`, `rsplit`,`확장자 추출(파일명다루기)`,`starswith`,`isdigit`,`isalpha`,`isalnum`)
- [[Python_String_Case_Replace]] : 대소문자 변환과 치환 (`upper`, `swapcase`, `replace` vs `re.sub`,`lower`,`re.search`)
- [[Python_Sorting_Logic]] : 딕셔너리 정렬과 sorted vs sort 차이 (`sorted`,`sort`)
- [[Python_Builtin_Functions]] : 코드를 줄여주는 마법, 내장 함수 (`max`, `min`, `sum`, `len`, `abs`,`ord`,`chr`)
- [[Python_Math_Module]] : 정밀한 수학 연산 (`math.prod`, `ceil`, `floor`, `gcd`)
- [[Python_Statistics_Module]] : 데이터 요약 통계 (`mean`, `median`, `stdev`)
- [[Python_Collections_Counter]] : 빈도 분석 (`Counter`, `most_common`)
- [[Python_Functions]] : 함수 정의와 인자 (`def`, `*args`, `**kwargs`)
- [[Python_Variable_Swapping]] : 파이썬 변수 교환 (`swap`)

## Level 2. 데이터 자료구조 (Data Structures)

> "데이터를 담는 그릇을 이해해요. (JSON 처리의 핵심)"

- [[Python_Lists_Tuples]] : 순서가 있는 데이터 (`[]`, `()`)
- [[Python_List_Comprehension]] (리스트 컴프리헨션 심화)
- [[Python_Dictionaries]] : 키-값 쌍 데이터 (`{}`, `.get()`,`.items()`,`.keys()`,`values()` ,`in`) 
-  [[Python_JSON]] : 딕셔너리와 문자열 변환 (`json.loads`, `json.dumps`)
- [[Python_Sets]] : 중복 제거하기 (`set`)

## Level 3. 모듈과 환경 (Modularization)

> "남이 짠 코드를 가져다 쓰고, 내 코드를 정리해요."

- [[Python_Modules_Imports]] : `import`와 `from`의 차이, 패키지 구조
- [[Python_Entry_Point]] : 프로그램 시작점과 실행 제어 (`if __name__ == '__main__':`)
- [[Python_Regex]] : 정규표현식 마스터 (`re.search` vs `match`, `findall`, `group`,`span`,`re.sub`)
- [[Python_Virtual_Env]] : 가상 환경 관리 (`venv`, `pip`, `requirements.txt`) - docker 실행시 필수 

## Level 4. 견고한 코드 만들기 (Robustness)

> "에러가 나도 파이프라인이 멈추지 않게 해요."

- [[Python_Error_Handling]] : 예외 처리 (`try`, `except`, `finally`)
- [[Python_File_IO]] : 파일 읽고 쓰기 (`open`, `with` 문)
- [[Python_Logging]] : `print` 대신 로그 남기기 (`logging`)
- [[Python_Requests_Methods]] : API 요청 보내기 (GET vs POST, `params`, `data`)
- [[Python_Requests_Response]] : API 응답 상태 확인과 데이터 추출 (`status_code`, `.json()`)
- [[HTTP_Status_Codes]] : 응답 코드 해석하기 (200, 404, 500 등)

## Level 5. 객체 지향 프로그래밍 (OOP)

> "Airflow 오퍼레이터가 왜 이렇게 생겼는지 이해해요."

- [[Python_Classes_Objects]] : 클래스와 속성 접근 (`Class`, `Instance`, `self`, `object.attribute` - `.value`의 정체!)
- [[Python_Inheritance]] : 상속과 오버라이딩 (`BaseOperator` 상속받기)

## Level 6. 고급 기능 (Advanced for DE)

> "Airflow 2.0의 마법 같은 기능을 써봐요."

- [[Python_Decorators]] : 골뱅이(`@`)의 정체 (TaskFlow API 핵심)
- **[[Python_Iterables_Iterators]]** : 하나씩 꺼내 쓰는 줄 세우기 (`iter`, `next`, `Iterable` vs `Iterator`)
- [[Python_Generators_Yield]] : 메모리를 아끼는 반복문 (`yield`, `next`, `generator`)
- [[Python_Lambda_Map]] : 이름 없는 함수와 고속 처리(`lambda`,`Map` ,`list` ,`filter` ,`reduce`)
- [[Python_DateTime]] : 날짜 계산과 실행 대기 (`datetime`, `timedelta`, `time.sleep`) - 스케줄링 필수!
- [[Python_Random_Seed]] : 랜덤 시드 (`random`,`seed`,`randint`)
- [[Python_AST_Structure_Analysis]] : 코드 구조 분석과 안전한 파싱 (`ast`, `literal_eval`, `parse`)
---
