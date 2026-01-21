---
aliases:
  - Airflow 베스트 프랙티스
  - DAG 설계 원칙
  - 원자성
  - 멱등성
  - One Task One Operator
tags:
  - Airflow
  - BestPractice
  - 면접질문
  - 고급
related:
  - "[[DAG_Concept]]"
  - "[[Idempotency(멱등성)]]"
---
## 개념 한 줄 요약

DAG를 짤 때 **"나중에 에러가 났을 때, 최소한의 부분만 재실행할 수 있도록"** 작업을 잘게 쪼개고(Atomicity), **"여러 번 실행해도 결과가 똑같도록"** 만드는(Idempotency) 설계 원칙 모음입니다.

## 왜 필요한가 (Why)

**문제 상황:**

하나의 Python 함수 안에 `Extract -> Transform -> Load` 로직을 다 때려 넣었다고 칩시다.
* 데이터는 다 가져왔는데(Extract 성공), DB에 넣다가(Load) 에러가 났습니다.
* 재시도(Retry)를 하려면 **이미 성공한 Extract부터 다시 해야 합니다.** (시간 낭비 + API 호출 비용 발생)

**해결책:**

역할별로 Operator를 쪼갭니다.
* `Extract Task` -> `Transform Task` -> `Load Task`
* 이렇게 하면 Load만 실패했을 때, **Load만 재시도**하면 됩니다. 이것이 Airflow를 쓰는 진짜 이유입니다.

## 1. One Operator = One Task (원자성)

> "하나의 오퍼레이터는 하나의 작업만 수행해야 한다."

### ❌ 나쁜 예 (Monolithic Task)
```python
# 하나의 함수가 너무 많은 일을 함 (실패하면 처음부터 다시 해야 함)
def etl_all_in_one():
    data = extract_from_api()  # 10분 걸림
    clean_data = transform(data)
    upload_to_db(clean_data)   # 여기서 에러나면 앞의 10분 날림

t1 = PythonOperator(task_id='etl_task', python_callable=etl_all_in_one)
```

### ✅ 좋은 예 (Atomic Tasks)

단계별로 쪼갭니다.

```python
# 1. 추출
t1 = PythonOperator(task_id='extract', python_callable=extract_from_api)
# 2. 변환
t2 = PythonOperator(task_id='transform', python_callable=transform)
# 3. 적재
t3 = PythonOperator(task_id='load', python_callable=upload_to_db)

t1 >> t2 >> t3  # 명확한 의존성 관리
```

---
## 멱등성 (Idempotency) 유지하기

"같은 DAG를 100번 재실행해도, DB에 쌓이는 데이터는 1건이어야 한다."

- **설명:** `INSERT INTO` 쿼리를 그냥 날리면, 재실행할 때마다 데이터가 중복으로 쌓입니다.
- **해결:** `DELETE` 후 `INSERT` (Overwrite) 방식을 쓰거나, `Upsert` (있으면 업데이트, 없으면 삽입) 방식을 써야 합니다.
- **원칙:** Airflow에서 재시도(Retry)는 언제든 일어날 수 있다는 가정하에 코드를 짜야 합니다.

---
## Top-level Code 제한하기 (성능 이슈)

> "DAG 파일의 맨 윗부분(함수 밖)에서 무거운 작업을 하지 마라."

- **문제:** Airflow 스케줄러는 DAG 폴더에 있는 파이썬 파일을 **매초마다** 읽어서 파싱합니다.
- **실수:** 파일 상단에 `Connect DB`나 `Requests.get` 같은 코드를 넣으면, 스케줄러가 그거 처리하느라 멈춰버립니다.
- **해결:** 무거운 작업은 반드시 **Operator 안이나 함수 안(`def ...`)** 에 넣어야 합니다.

---
## 초보자가 자주 착각하는 포인트

1. **"쪼개면 코드가 너무 길어지는데요?"**
    - 네, 코드는 길어지지만 **운영 비용(야근 시간)** 은 줄어듭니다.
    - Airflow는 '코드의 간결함'보다 **'운영의 안정성'** 을 위해 쓰는 도구임을 명심하세요.
        
2. **"데이터는 어떻게 넘겨요?"**
    - 작업을 쪼개면 변수를 직접 못 넘깁니다. 그래서 **XCom**이나 **S3/DB** 같은 중간 저장소를 써야 합니다. (이건 `Data Passing` 파트에서 배웁니다.)