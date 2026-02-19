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
linked:
  - file:////Users/gong/gong_study_de/project/generator/fake_log_producer.py
  - file:////Users/gong/gong_study_de/project/pyflink_app/real_time_aggregation.py
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