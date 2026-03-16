---
aliases:
  - Airflow센서
  - Sensor
  - 외부이벤트감지
  - Poke모드
tags:
  - Airflow
related:
  - "[[Airflow_Operators]]"
  - "[[Task_Dependencies]]"
  - "[[Branching]]"
  - "[[Pandas_Json_Normalize]]"
---
## 개념 한 줄 요약

Airflow의 **Sensor**는 특정 **조건이 충족될 때까지(Condition is met) 태스크 실행을 잠시 멈추고 기다리는(Pauses execution)** 특수한 형태의 오퍼레이터입니다.

---
## 왜 필요한가 (Why)

**문제점:**
- 데이터 파이프라인은 혼자 도는 게 아닙니다. "S3에 파일이 올라와야 처리를 시작하는데?", "옆 팀의 API가 준비되어야 호출하는데?" 같은 **외부 의존성**이 존재합니다.
- 파일이 아직 없다고 바로 에러를 내고 죽어버리면(Fail), 운영자가 매번 다시 켜줘야 하는 번거로움이 생깁니다.

**해결책:**
- Sensor를 문지기처럼 세워두고 **"파일 들어왔어? (Poke)"** 라고 주기적으로 확인하게 합니다. 조건이 만족되면 문을 열어 다음 태스크(Downstream)가 실행되도록 합니다.

---
## Practical Context (실무 활용)

1.  **FileSensor / S3KeySensor**: "고객사가 FTP나 S3에 매출 데이터를 올려주면 그때 크롤링 시작해라."
2.  **SqlSensor**: "DB 테이블에 오늘 날짜 데이터가 1건이라도 들어오면 통계 작업을 시작해라."
3.  **ExternalTaskSensor**: "A라는 DAG가 다 끝나면 B라는 DAG를 실행해라."
4.  **TimeSensor**: "새벽 3시까지는 무조건 대기해라."

---
## Code Core Points (핵심 파라미터)

센서를 쓸 때 가장 중요한 건 **"어떻게 기다릴 것인가(Mode)"** 를 정하는 것입니다.

1.  **`mode='poke'`**:
    - 계속 자리를 차지하고 앉아서 "왔어? 왔어?" 하고 물어봅니다.
    - 반응은 빠르지만, **워커(Worker) 슬롯 하나를 계속 점유**하므로 대기 시간이 길어지면 자원 낭비가 심합니다.
2.  **`mode='reschedule'` (Default 권장)**:
    - 한번 확인하고 없으면 **자리를 비켜줍니다(Slot 해제)**.
    - 정해진 시간(`poke_interval`) 뒤에 다시 스케줄링되어 확인합니다. 자원을 효율적으로 씁니다.
3.  **`poke_interval`**: 확인하는 주기(초 단위). "30초마다 확인해"
4.  **`timeout`**: "1시간 동안 안 오면 그냥 에러 내고 끝내라." (무한 대기 방지)

---
## Detailed Analysis (코드 해석)

`SqlSensor`를 사용하여 특정 데이터가 들어왔는지 감시하는 예제입니다.

### import 하는 법 

```python
# ❌ Airflow 1.x 방식
from airflow.sensors.sql import SqlSensor

# ✅ Airflow 2.x 방식
from airflow.providers.common.sql.sensors.sql import SqlSensor
```

### 코드 

```python
sql_sensor = SqlSensor(
    task_id='wait_for_data',
    conn_id='my_postgres_connection', # 확인할 DB 접속 정보
    # 조건문: 이 SQL 결과가 True(또는 데이터 있음)일 때까지 기다림
    sql="SELECT COUNT(*) FROM sample_table WHERE key='hello'",
    mode='poke',         # 계속 자리를 지키며 확인 (짧은 대기 시간용)
    poke_interval=30,    # 30초마다 한 번씩 쿼리를 날려봄
    timeout=600,         # (추가) 600초(10분) 지나도 안 오면 Fail 처리
)
```

---
## 초보자가 자주 착각하는 포인트(주의사항)

- **무조건 `poke` 모드 사용 금지**:
    - 센서가 100개인데 워커가 10개뿐이라면? `poke` 모드로 된 센서 10개가 돌기 시작하면 워커가 꽉 차서(Deadlock), 실제 일을 해야 할 다른 태스크들이 영원히 실행되지 못하는 참사가 발생합니다. 
    - **대기 시간이 길 것 같으면 무조건 `reschedule`을 쓰세요.**
 
- **센서는 만능이 아니다**:
    - 센서는 '감지'만 할 뿐, 데이터를 가져오지는 않습니다. 데이터를 가져오려면 센서 다음에 `PythonOperator` 등을 연결해야 합니다.
