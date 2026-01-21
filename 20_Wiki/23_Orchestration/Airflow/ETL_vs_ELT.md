---
aliases:
  - ETL과 ELT 차이
  - 데이터 파이프라인 패턴
tags:
  - 아키텍처
  - 개념
  - 면접질문
related:
  - "[[00_Airflow_HomPage]]"
  - "[[Airflow_Architecture(아키텍처)]]"
---
## 개념 한 줄 요약

데이터를 목적지에 넣기 전에 가공하면 **ETL** (요리해서 냉장고 넣기), 
일단 넣고 나서 나중에 가공하면 **ELT** (장 봐온 거 몽땅 냉장고에 넣고 꺼내서 요리하기)입니다.

---
## 왜 필요한가 (Why)

**문제 상황 (ETL의 한계):**

과거에는 저장 공간이 비쌌습니다. 그래서 필요 없는 데이터를 미리 버리고, 압축해서 DB에 넣어야 했습니다.
그런데 나중에 기획자가 **"아, 아까 버린 그 로그 데이터도 다시 필요한데요?"** 라고 하면, 엔지니어는 소스 시스템부터 다시 데이터를 긁어와야 하는 대공사가 벌어집니다.

**해결책 (ELT의 등장):**

클라우드 저장소(S3, BigQuery)가 엄청나게 싸고 빨라졌습니다.
**"야, 굳이 미리 고생해서 줄이지 마. 그냥 원본(Raw) 그대로 다 넣어. 나중에 SQL로 가공하면 되잖아."**
이게 ELT입니다. 원본이 저장소에 다 있으니, 분석 요구사항이 바뀌어도 SQL만 다시 돌리면 됩니다.

---
## 실무 맥락에서의 사용 이유

**"Airflow로 복잡한 파이썬 코드 짜지 말고, 그냥 쿼리(SQL)만 날리세요."**
최근 데이터 팀의 트렌드입니다.
* **ETL:** Airflow 워커가 끙끙대며 데이터를 메모리에 올려서 가공함. (워커 터짐, 느림)
* **ELT:** Airflow는 "BigQuery야, 이 쿼리 좀 돌려"라고 명령만 하고 쉼. 실제 연산은 짱짱한 BigQuery가 처리함. (빠름, 안정적)

## 상세 분석 (ETL vs ELT 비교)

| 구분 | ETL (Extract -> Transform -> Load) | ELT (Extract -> Load -> Transform) |
| :--- | :--- | :--- |
| **순서** | 추출 -> **변형(Python/Spark)** -> 적재 | 추출 -> **적재(DB/Lake)** -> **변형(SQL)** |
| **주요 도구** | Spark, Python Pandas, Informatica | BigQuery, Snowflake, dbt |
| **장점** | 민감 정보(개인정보)를 미리 지우고 넣을 수 있어 보안에 좋음. | 원본 데이터가 보존됨. 유연성이 매우 높음. |
| **단점** | 로직 수정 시 처음부터 다시 해야 함. 파이프라인 구축이 오래 걸림. | 불필요한 데이터까지 다 저장해서 저장 비용이 조금 더 듦. |

## 코드 핵심 포인트 (Airflow에서의 차이)

### 1. ETL 방식 (PythonOperator 사용)

```python
# Airflow가 직접 데이터를 씹고 뜯고 맛보고 즐김 (무거움)
def process_data():
    df = pd.read_csv("data.csv")
    df = df[df['price'] > 1000]  # 변형
    df.to_sql("target_table")    # 적재

task = PythonOperator(task_id='etl_task', python_callable=process_data)
```

### 2. ELT 방식 (BigQueryOperator 사용)

```python
# Airflow는 명령만 내림. 실제 힘든 일은 BigQuery가 함 (가벼움)
task = BigQueryInsertJobOperator(
    task_id='elt_task',
    configuration={
        "query": {
            "query": "INSERT INTO target SELECT * FROM raw WHERE price > 1000",
            "useLegacySql": False,
        }
    }
)
```

---
## 초보자가 자주 착각하는 포인트

1. **"ELT가 무조건 정답인가요?"**
    - 아닙니다. **개인정보(주민번호, 비번)** 는 절대 원본 그대로 DB에 올리면 안 됩니다. 이런 건 앞단에서 마스킹 처리를 하는 ETL 과정이 반드시 필요합니다.
        
2. **"ELT 하면 Airflow 필요 없나요?"**
    - 여전히 필요합니다! "언제 Load 하고, 언제 Transform(SQL) 할지" 순서를 정해주는 **지휘자** 역할은 Airflow가 가장 잘하기 때문입니다. 다만 Airflow가 '일꾼'에서 '감독관'으로 역할이 바뀔 뿐입니다.
