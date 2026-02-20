---
aliases:
tags:
  - Project
related:
  - "[[00_Project_HomePage]]"
  - "[[PyFlink_Table_API_Intro]]"
  - "[[PyFlink_Table_Expressions]]"
  - "[[PyFlink_SQL_Windows]]"
  - "[[PyFlink_SQL_Watermark]]"
  - "[[PyFlink_SQL_Integration]]"
  - "[[PyFlink + Kafka 연동 완벽 가이드 ⭐️]]"
  - "[[Streamlit_Charts]]"
  - "[[Streamlit_Concept]]"
  - "[[Pandas_Selection]]"
  - "[[Pandas_Read_Write]]"
  - "[[Throttling_RateLimiting]]"
linked:
  - file:////Users/gong/gong_study_de/project/generator/fake_log_producer.py
  - file:////Users/gong/gong_study_de/project/pyflink_app/real_time_aggregation.py
  - file:////Users/gong/gong_study_de/project/streamlit_app/Dashboard.py
---
# Kafka데이터 > 1분마다 매출 합계 구하는 Flink



```python
from pyflink.table import EnvironmentSettings, TableEnvironment
from pyflink.table.window import Tumble
from pyflink.table.expressions import col, lit

def real_time_aggregation():
    # 1. 환경 설정: "이 작업은 한 번 하고 끝나는 게 아니라 계속 도는 스트리밍(Streaming) 방식이야!"
    settings = EnvironmentSettings.in_streaming_mode()
    t_env = TableEnvironment.create(settings)
    
    print(">>> PyFlink Start!!")
    
    # 2. Source Table 생성 (데이터를 빨아들이는 입구)
    # 실제 DB에 테이블을 만드는 게 아니라, Kafka의 특정 토픽을 마치 DB 테이블처럼 맵핑하는 겁니다.
    t_env.execute_sql("""
        CREATE TABLE source_table(
            order_id STRING,
            total_amount BIGINT,
            category STRING,
            
            -- 이벤트가 실제로 발생한 시간 (JSON 안의 시간 필드)
            `timestamp` TIMESTAMP(3),
            
            -- [핵심] 워터마크: 늦게 오는 데이터를 5초까지는 봐줄게!
            -- 예: 12시 01분 05초가 되어서야 "아, 이제 12시 01분 00초까지의 1분치 윈도우를 닫고 계산해야겠다"라고 판단함.
            WATERMARK FOR `timestamp` AS `timestamp` - INTERVAL '5' SECOND
        ) WITH(
            'connector'='kafka',                                 -- 어디서? 카프카에서
            'topic'='user-log',                                  -- 어떤 토픽? user-log
            'properties.bootstrap.servers'='kafka:9092',         -- 카프카 브로커 주소
            'properties.group.id'='project-flik-aggregator-dev', -- 컨슈머 그룹 ID
            'scan.startup.mode'='latest-offset',                 -- [중요] 프로그램 '실행된 이후'의 새 데이터만 읽겠다!
            'format'='json',                                     -- 들어오는 데이터 형태는 JSON
            'json.timestamp-format.standard' = 'ISO-8601'        -- 시간 포맷 지정
        )
    """)
    
    # 3. 집계 쿼리 작성 (어떻게 요리할 것인가)
    # TUMBLE: 텀블링 윈도우 (시간이 겹치지 않고 뚝뚝 끊어지는 윈도우)
    agg_query="""
        SELECT
            window_start,                  -- 윈도우 시작 시간 (예: 12:00:00)
            window_end,                    -- 윈도우 종료 시간 (예: 12:01:00)
            category,                      -- 그룹 기준 1: 카테고리
            SUM(total_amount) as total_sales,   -- 총 매출액 집계
            COUNT(order_id) as total_orders     -- 총 주문 건수 집계
        FROM TABLE(
            -- source_table을 가져와서, timestamp 기준으로 1분(1 MINUTE) 단위로 잘라라!
            TUMBLE(TABLE source_table, DESCRIPTOR(`timestamp`), INTERVAL '1' MINUTE)
        )
        -- 잘라진 네모 박스 안에서 다시 카테고리별로 그룹핑
        GROUP BY window_start, window_end, category
    """
        
    # 4. Sink Table 생성 (데이터를 뱉어내는 출구)
    # 연산 결과를 콘솔 화면(터미널)에 출력(print)하기 위한 가상의 테이블입니다.
    t_env.execute_sql("""
        CREATE TABLE print_sink(
            window_start TIMESTAMP(3),
            window_end TIMESTAMP(3),
            category STRING,
            total_sales BIGINT,
            total_orders BIGINT
        ) WITH(
            'connector'='print'  -- 화면에 찍어줘! (실무에선 여기를 jdbc, kafka, s3 등으로 바꿈)
        )
    """)
    
    # 5. 실행 (파이프라인 연결)
    # "agg_query로 계산한 결과를 print_sink에 계속 밀어 넣어라(INSERT)!"
    t_env.execute_sql(f"INSERT INTO print_sink {agg_query}")
    
if __name__=='__main__':
    real_time_aggregation()
```


---
### 실행 순서

이제 Flink JobManager 컨테이너 안으로 들어가서 작업을 제출(Submit) 해봅시다.


**① 데이터 발생시키기 (터미널 새 탭)** 

Flink는 데이터가 들어와야 작동합니다.= producer 실행 제일먼저!

```bash
python3 generator/fake_log_producer.py
```


**② Flink Job 제출**

```bash
# jobmanager 컨테이너 이름 확인 (docker ps) 후 입력
# 보통 project-jobmanager-1 입니다.
docker exec -it project-jobmanager-1 \
  ./bin/flink run \
  -py /opt/flink/pyflink_app/real_time_aggregation.py \
  -j /opt/flink/lib/flink-sql-connector-kafka-3.1.0-1.18.jar
```


**③ 결과 확인** 

Flink는 `print` 커넥터를 쓰면 **TaskManager의 로그**에 결과를 출력합니다.

```bash
docker logs -f project-taskmanager-1
```

### 결과가 어떻게 나올까요?

1분 정도 기다리면(데이터가 쌓이면) 로그창에 이런 식으로 뜰 겁니다.

```text
+I[2026-02-19T23:53, 2026-02-19T23:54, 생활, 4456800, 9]
+I[2026-02-19T23:53, 2026-02-19T23:54, 스포츠, 10149100, 15]
+I[2026-02-19T23:53, 2026-02-19T23:54, 식품, 5661100, 6]
+I[2026-02-19T23:53, 2026-02-19T23:54, 패션, 6212500, 9]
+I[2026-02-19T23:53, 2026-02-19T23:54, 전자, 5584700, 6]
```

| 기호      | 의미                     |
| ------- | ---------------------- |
| `+I`    | Insert — 새로 집계된 윈도우 결과 |
| 첫 번째 시간 | 윈도우 시작 시각              |
| 두 번째 시간 | 윈도우 종료 시각              |
| 이후 값    | 카테고리 / 매출 합계 / 주문 건수   |

----
# 결과 화면을 PostgreSQL에 저장하기

## DB에 테이블 만들기

Flink는 테이블에 데이터를 꽂아주기만 할 뿐, 테이블 자체를 생성해주지는 않습니다. 
먼저 접속하려는 PostgreSQL 데이터베이스(예: `airflow`)에 접속하여 아래 쿼리로 그릇을 만들어 줍니다.
이때 `localhost:5433/airflow` 로 docker compose.yml에다가  지정해놨기때문 

```yml
postgres:

image: postgres:13

ports:

- "5433:5432"

environment:

POSTGRES_USER: airflow

POSTGRES_PASSWORD: airflow

POSTGRES_DB: airflow
```

### DataGrip 에 생성

```sql
CREATE TABLE sales_aggregation (
    window_start TIMESTAMP,
    window_end TIMESTAMP,
    category VARCHAR(255),
    total_sales BIGINT,
    total_orders BIGINT
);
```

##  필수 JAR 파일(통역사) 설치하기

Flink가 DB와 소통하려면 **2개의 JAR 파일**(JDBC 커넥터, DB 드라이버)이 Flink의 `JobManager`와 `TaskManager` 양쪽 모두에 설치되어 있어야 합니다. 
터미널을 열고 아래 명령어를 순서대로 실행해 다운로드 후 재시작합니다.

```bash
# JobManager에 다운로드
docker exec -it -u root project-jobmanager-1 curl -fL -o /opt/flink/lib/flink-connector-jdbc-3.1.2-1.18.jar https://repo1.maven.org/maven2/org/apache/flink/flink-connector-jdbc/3.1.2-1.18/flink-connector-jdbc-3.1.2-1.18.jar
docker exec -it -u root project-jobmanager-1 curl -fL -o /opt/flink/lib/postgresql-42.6.0.jar https://repo1.maven.org/maven2/org/postgresql/postgresql/42.6.0/postgresql-42.6.0.jar

# TaskManager(실제 일꾼)에 다운로드
docker exec -it -u root project-taskmanager-1 curl -fL -o /opt/flink/lib/flink-connector-jdbc-3.1.2-1.18.jar https://repo1.maven.org/maven2/org/apache/flink/flink-connector-jdbc/3.1.2-1.18/flink-connector-jdbc-3.1.2-1.18.jar
docker exec -it -u root project-taskmanager-1 curl -fL -o /opt/flink/lib/postgresql-42.6.0.jar https://repo1.maven.org/maven2/org/postgresql/postgresql/42.6.0/postgresql-42.6.0.jar

# 설치된 JAR 파일을 인식하도록 Flink 컨테이너 재시작
docker restart project-jobmanager-1 project-taskmanager-1
```

(주의: `docker-compose down`을 하면 파일이 날아가므로 반드시 `restart`만 해야 합니다.)

## 파이썬 코드 수정

```python
# 4. Sink Table 생성 (데이터를 뱉어내는 출구 - DB 연동)
    t_env.execute_sql("""
        CREATE TABLE jdbc_sink (
            window_start TIMESTAMP(3),
            window_end TIMESTAMP(3),
            category STRING,
            total_sales BIGINT,
            total_orders BIGINT
        ) WITH (
            'connector'='jdbc',
            --  중요: 내 컴퓨터(localhost)가 아니라 도커 서비스 이름(postgres)을 사용!
            'url'='jdbc:postgresql://postgres:5432/airflow',
            'table-name'='sales_aggregation',
            'username'='airflow',    -- 실제 DB 계정
            'password'='airflow',    -- 실제 DB 비밀번호
            'driver'='org.postgresql.Driver'
        )
    """)
    
    # 5. 실행 (파이프라인 연결)
    # 수정 전: t_env.execute_sql(f"INSERT INTO print_sink {agg_query}")
    # ✅ 수정 후:
    t_env.execute_sql(f"INSERT INTO jdbc_sink {agg_query}")
```

## 대시보드 추가 (`Dashboard.py`)

```python
import pandas as pd
import plotly.express as px
from sqlalchemy import create_engine

# ---------------------------------------------------------
# 1. DB 연결 (루프 바깥에서 한 번만 선언하는 것이 성능에 좋습니다)
# ---------------------------------------------------------
flink_url = "postgresql://airflow:airflow@postgres:5432/airflow"
flink_engine = create_engine(flink_url)

# 카운터 초기화
loop_count = 0 

# ---------------------------------------------------------
# 2. 실시간 데이터 수신 루프 (Kafka Consumer 등)
# ---------------------------------------------------------
for message in consumer:
    # (여기에 기존 Kafka 실시간 처리 코드 작성) ...
    
    loop_count += 1  # 메시지가 들어올 때마다 1씩 증가
    
    # ---------------------------------------------------------
    # 3. [핵심] 20번 루프가 돌 때마다 한 번씩 Flink 집계 DB 조회
    # ---------------------------------------------------------
    
    # time.sleep(0.05)로 1번 도는데 걸리는 시간 0.05초 
    # 20번 도는데 걸리는 시간= 1초(0.05 x 20=1.0) 그래서 1초에 한번만 갱신
    if loop_count % 20 == 0:
        try:
            # 가장 최근 시간(MAX window_end)의 1분치 데이터만 가져오는 쿼리
            flink_query = """
                SELECT category, total_sales, total_orders, window_end
                FROM sales_aggregation
                WHERE window_end = (SELECT MAX(window_end) FROM sales_aggregation)
                ORDER BY total_sales DESC
            """
            
            # 엔진을 이용해 DB에서 데이터 읽어오기
            df_flink = pd.read_sql(flink_query, flink_engine)
        
            # 데이터가 존재할 경우 화면 업데이트
            if not df_flink.empty:
                # ① 테이블 UI 업데이트
                flink_table_slot.dataframe(df_flink, use_container_width=True, hide_index=True)
                
                # ② 도넛 차트 UI 업데이트
                latest_time = df_flink['window_end'].iloc[0].strftime('%H:%M:%S')
                fig_flink = px.pie(
                    df_flink,
                    names='category',
                    values='total_sales',
                    title=f"최근 1분 매출 비율 ({latest_time} 기준)",
                    hole=0.4,
                    color='category'
                )
                flink_chart_slot.plotly_chart(fig_flink, use_container_width=True,key=f"flink_pie_{loop_count}")
        
        except Exception as e:
            flink_table_slot.warning(f"Flink대기중...{e}")
    time.sleep(0.05)
```


## 실행 결과 확인 

**① 데이터 발생시키기 (터미널 새 탭)** 

Flink는 데이터가 들어와야 작동합니다.= producer 실행 제일먼저!

```bash
python3 generator/fake_log_producer.py
```

**② Flink Job 제출**

```bash
docker exec -it project-jobmanager-1 \
  ./bin/flink run \
  -py /opt/flink/pyflink_app/real_time_aggregation.py
```

>1분마다 DB에 실시간 집계 데이터가 쑥쑥 들어오는 것을 확인할 수 있었다!!!

![[스크린샷 2026-02-20 오후 2.18.34.png| 700x500]]

**터미널 3** 대시보드 확인

```bash
python3 -m streamlit run streamlit_app/Dashboard.py
```

![[스크린샷 2026-02-20 오후 3.12.28.png|720x400]]
