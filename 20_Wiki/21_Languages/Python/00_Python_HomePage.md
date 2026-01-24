
## Level 1. 기초 문법 (Syntax)

> "파이썬과 대화하는 법을 배워요."

- [[Python_Variables_Types]] : 변수와 자료형 (String, Int, Bool)
- [[Python_Control_Flow]] : 흐름 제어하기 (If, For, While)
- [[Python_Functions]] : 함수 정의와 인자 (`def`, `*args`, `**kwargs` - Airflow 필수!)

## Level 2. 데이터 자료구조 (Data Structures)

> "데이터를 담는 그릇을 이해해요. (JSON 처리의 핵심)"

- [[Python_Lists_Tuples]] : 순서가 있는 데이터 (`[]`, `()`)
- [[Python_Dictionaries]] : 키-값 쌍 데이터 (`{}`) - **가장 중요!**
-  [[Python_JSON]] : 딕셔너리와 문자열 변환 (`json.loads`, `json.dumps`)
- [[Python_Sets]] : 중복 제거하기 (`set`)

## Level 3. 모듈과 환경 (Modularization)

> "남이 짠 코드를 가져다 쓰고, 내 코드를 정리해요."

- [[Python_Modules_Imports]] : `import`와 `from`의 차이, 패키지 구조
- [[Python_Virtual_Env]] : 가상 환경 관리 (`venv`, `pip`, `requirements.txt`)

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

- [[Python_Classes_Objects]] : 붕어빵 틀과 붕어빵 (Class & Instance)
- [[Python_Inheritance]] : 상속과 오버라이딩 (`BaseOperator` 상속받기)

## Level 6. 고급 기능 (Advanced for DE)

> "Airflow 2.0의 마법 같은 기능을 써봐요."

- [[Python_Decorators]] : 골뱅이(`@`)의 정체 (TaskFlow API 핵심)
- [[Python_Lambda_Map]] : 이름 없는 함수와 고속 처리
- [[Python_DateTime]] : 날짜 계산과 실행 대기 (`datetime`, `timedelta`, `time.sleep`) - 스케줄링 필수!

---

## Practical Context (사용 가이드)

1.  **Airflow와의 연계:**
    * **Level 1 (`**kwargs`)**: Airflow의 `context` 변수를 받을 때 무조건 쓰입니다.
    * **Level 5 (Class)**: 나만의 커스텀 오퍼레이터를 만들 때 필수입니다.
    * **Level 6 (Decorator)**: 요즘 Airflow는 `with DAG...` 대신 `@dag`, `@task`를 많이 씁니다.
2.  **학습 순서:** Level 1, 2를 빠르게 떼고, **Level 3(환경 설정)** 와 **Level 6(날짜 처리)** 를 먼저 보는 것도 전략입니다.

## Common Beginner Misconceptions

* **"들여쓰기(Indentation) 대충 해도 되나요?"**: 파이썬에서 들여쓰기는 문법 그 자체입니다. 탭(Tab)과 스페이스바를 섞어 쓰면 에러 지옥을 맛보게 됩니다. (스페이스 4칸 권장)
* **"클래스(Class) 몰라도 되나요?"**: 스크립트 짤 때는 몰라도 되지만, **Airflow의 작동 원리**를 이해하려면 클래스와 상속 개념은 꼭 넘어야 할 산입니다.