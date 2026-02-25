---
aliases:
  - Python 날짜
  - 시간
  - time 모듈
  - datetime
  - sleep
  - timedelta
  - isoformat
  - date
  - today
  - isocalendar
  - weekday
tags:
  - Python
related:
  - "[[Blocking_vs_NonBlocking]]"
  - "[[Airflow_Queues_Scaling]]"
  - "[[Python_Modules_Imports]]"
  - "[[Airflow_Sensors]]"
  - "[[Python_Mock_Data_Generator]]"
  - "[[Python_Calendar_Module]]"
  - "[[00_Python_HomePage]]"
---

## 개념 한 줄 요약

데이터 엔지니어링에서 **시간(Time)** 은 두 가지 의미를 가져요.

1.  **시점 (Point in Time):** "2026년 1월 23일 14시" (로그 찍기, 스케줄링) -> `datetime`, `date` 모듈
2.  **기간/대기 (Duration/Pause):** "10초 동안 멈춰" (API 호출 간격, 폴링) -> `time` 모듈

---
## 핵심 개념 3가지 

1. **datetime (시계+달력):** "지금 몇 년 몇 월 며칠, 몇 시 몇 분이야?" (날짜와 시간 모두)
2. **date (달력):** "오늘 며칠이야?" (시간 빼고 **날짜만**)
3. **timedelta (스톱워치/기간):** "3일 걸려", "10분 차이야" (시간의 **간격**)
4. **time.sleep (일시정지):** "잠깐 멈춰!" (프로그램 **대기**)

---
## 왜 필요한가 (Why)

왜 그냥 문자열 `"2024-01-01"`을 안 쓰고 귀찮게 이런 걸 쓸까?

**문제점:**
- 컴퓨터 입장에서 `"2024-01-01"`은 그냥 글자(String)일 뿐이야. 
- 여기서 내일 날짜를 구하려고 `"2024-01-01" + 1`을 하면 에러가 나. (글자에 숫자를 더할 순 없으니까!)

**해결책:**
- 파이썬이 이해할 수 있는 객체로 만들어야 "날짜 더하기/빼기", "무슨 요일인지 찾기" 같은 **시간 계산**이 가능해져.

---
## Practical Context (실무 활용)

**Airflow 스케줄링 & 파티셔닝:**
- 어제 날짜(`now - timedelta(days=1)`)의 데이터를 가져와서, 문자열(`strftime('%Y-%m-%d')`)로 변환해 폴더명(`ds=2026-02-23`)을 만들 때 숨 쉬듯 사용합니다.

**API Rate Limit 대응:**
- 외부 API가 "1초에 5번만 호출 가능" 제한이 있다면, `time.sleep(0.2)`를 넣어 차단을 방지합니다.

**폴링(Polling) 구조:**
- "파일이 들어왔나?" 확인하고 없으면 `time.sleep(60)` 하고 다시 확인하는 구조의 핵심입니다.
- 단, `time.sleep()`은 CPU를 통째로 멈추는 **Blocking** 함수이므로 주의해야 합니다. (자세한 내용은 [[Blocking_vs_NonBlocking]] 참고)

**Timezone (UTC):**
-  글로벌 서비스나 데이터 파이프라인의 서버 시간은 무조건 **UTC(협정 세계시)** 로 맞추는 것이 국룰입니다. 
- 한국 시간(KST)은 화면에 보여줄 때만 변환합니다.

---
## 날짜만 깔끔하게 다루기 (date)

- 시간(시:분:초) 정보가 아예 필요 없고 "연-월-일" 달력 정보만 필요할 때 씁니다.

### 오늘 날짜와 특정 날짜 만들기

- `date.today()` -> 실행하는 시점의 오늘 날짜를 가져옵니다.
- `date(연, 월, 일)` -> 내가 원하는 특정 날짜 객체를 직접 만듭니다.

```python
from datetime import date

# 1. 오늘 날짜 가져오기 (시간 정보 없음)
today = date.today() 
print(today) # 출력: 2026-02-24

# 2. 특정 날짜 직접 만들기 
d_day = date(2026, 12, 25)
```

### 요일과 주차(Week) 구하기

-  `날짜객체.weekday()` -> 월요일(0)부터 일요일(6)까지 숫자로 요일을 알려줍니다.
- `날짜객체.isocalendar()` -> 국제 표준 기준으로 `(연도, 주차, 요일)` 3가지를 튜플로 묶어서 반환합니다. 
	 ->여기서 두 번째 값(`[1]`)을 빼오면 "올해의 몇 번째 주(Week)"인지 바로 알 수 있습니다.

```python
# 3. 무슨 요일인지 확인하기 (월=0, 화=1, ... 일=6) 
print(today.weekday()) # 출력 예: 1 (화요일) 

# 4. 올해의 몇 주차(Week)인지 확인하기 (주간 통계 낼 때 필수!) 
# isocalendar()는 (2026, 9, 2) 처럼 (연도, 주차, 요일)을 반환함 
year, week, weekday = today.isocalendar() 
print(f"오늘은 {year}년의 {week}번째 주입니다.")
```

----
## 시점과 기간 (datetime & timedelta)

- **`datetime`:** "오늘 며칠이지?" (특정 시점)
- **`timedelta`:** "하루(24시간)라는 시간의 양" (시간의 덩어리/간격) -> Time(시간) + Delta(차이)
	- `timedelta(days=1)`은 **"하루(24시간)라는 시간의 양"**
- **공식:** `시점(datetime) + 기간(timedelta) = 새로운 시점(datetime)`


```python
from datetime import datetime 

#  현재 시간 가져오기
now = datetime.now() 
print(now) 
# 출력: 2026-01-23 14:30:00.123456 (년-월-일 시:분:초.마이크로초)

# "하루"라는 시간 덩어리를 만든다.
one_day = timedelta(days=1) 

# "30분"이라는 시간 덩어리를 만든다.
thirty_mins = timedelta(minutes=30)

#  날짜 연산 (어제 날짜 구하기) -> Airflow 필수 로직!
yesterday = now - timedelta(days=1)

# UTC(협정 세계시) 기준 현재 시간 가져오기 (끝에 +00:00이 붙음)
now_utc = datetime.now(timezone.utc)
print(now_utc.isoformat())
# 결과: '2024-02-13T07:30:45.123456+00:00'
```

---
##  프로그램 대기 (time.sleep)

- **문법:** `time.sleep(초)`
- **동작 원리:** 지정된 초(Seconds) 만큼 프로그램을 멈춥니다(Blocking I/O).

>API 호출 제한을 피하거나 폴링할 때 씁니다. 하지만 너무 길게 잡으면 프로그램이 멈춥니다.
>컴퓨터는 너무 빨라서 반복문을 돌리면 0.001초 만에 끝납니다. 
>**"사람 속도"** 나 **"센서 주기"** 를 흉내 내려면 반드시 재워야 합니다.
>[[Blocking_vs_NonBlocking]] 문서를 참고

```python
import time

print("API 호출 시작...")
time.sleep(3)  # 3초 동안 아무것도 안 하고 대기 (CPU 멈춤)
print("3초 뒤 호출 완료!")
```

---
## 날짜 ➡️ 문자열 (Formatting / DB 저장용)

> "DB나 Kafka에 넣으려면 글자(String)로 바꿔야 해!"

- **`isoformat()`:** 국제 표준(ISO 8601)으로 바꿉니다. 중간에 `T`가 들어가며 시간 순서 정렬이 보장되어 빅데이터 시스템에서 가장 선호합니다.
- **`strftime(포맷)`:** **Str**ing **F**ormat **Time**. 객체(now)에게 특정 포맷의 글자로 변신하라고 시킵니다.
-  **`f-string`:** 최신 트렌드로 문자열 안에서 바로 포맷을 지정합니다.


```python
from datetime import datetime
now = datetime.now() 

# 1. isoformat (국제 표준 - 가장 추천)
print(now.isoformat()) # 결과: '2026-02-24T21:06:32.123456' <- 중간에 T가 들어감 

# 2. strftime (원하는 모양으로)
date_str = now.strftime("%Y-%m-%d") 
print(date_str) # 결과: "2026-02-24" (이제 더하기 빼기 불가능한 단순 글자!)

# 3. f-string (가장 간편함)
print(f"현재 시간: {now:%Y-%m-%d %H:%M}") # 결과: 현재 시간: 2026-02-24 21:06
```

---
## 문자열 ➡️ 날짜 (Parsing / 계산용)

- **문법:** `datetime.strptime(문자열, 포맷)`
- **의미:** **Str**ing **P**arse **Time**. 글자를 분석해서 계산 가능한 시간 객체로 만듭니다.
- **팁: `.date()`의 역할:** 시간(시:분:초)은 다 떼어버리고 오직 **연-월-일(날짜)** 정보만 남기고 싶을 때 객체 뒤에 `.date()`를 붙여서 꼬리를 자릅니다.

```python
from datetime import datetime

date_text = "2026/01/23" # (단순 글자 상태)

# 공장(datetime)한테 재료(글자, 순서규칙)를 주고 만들어 달라고 함
date_obj = datetime.strptime(date_text, "%Y/%m/%d")

print(date_obj)        # 출력: 2026-01-23 00:00:00 (시간이 00시로 자동 세팅됨)
print(date_obj.date()) # 출력: 2026-01-23 (시간 꼬리를 자르고 날짜만 추출!)
```

### 자주 쓰는 포맷 코드표 (Cheatsheet)

이거 외우지 말고 필요할 때마다 와서 보고 베껴!

|**코드**|**의미**|**예시 (2026년 1월 5일 오후 2시 30분)**|
|---|---|---|
|`%Y`|4자리 연도|`2026`|
|`%m`|2자리 월|`01` (0 채워짐)|
|`%d`|2자리 일|`05` (0 채워짐)|
|`%H`|24시간제|`14`|
|`%M`|분|`30`|
|`%S`|초|`00`|
|`%a`|요일 (짧게)|`Mon`, `Tue`|

----
## 초보자가 자주 착각하는 포인트

1. `import datetime` vs `from datetime import datetime`:
	- 파이썬의 `datetime` **모듈(파일)** 안에 `datetime`이라는 **클래스(도구)** 가 또 들어있어서 이름이 겹쳐.
	- 그래서 보통 `from datetime import datetime`이라고 써야 `datetime.now()`처럼 바로 쓸 수 있어. 그냥 `import datetime` 하면 `datetime.datetime.now()`라고 길게 써야 해서 귀찮아.

2. **문자열이랑 헷갈리지 말기:**
	- `"2024-01-01"` (따옴표 있음) ➡ **문자열(String)**. 계산 불가. ❌
	- `datetime(2024, 1, 1)` (따옴표 없음) ➡ **객체(Object)**. 계산 가능. ⭕️