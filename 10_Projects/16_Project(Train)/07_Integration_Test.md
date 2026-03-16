---
aliases:
  - 통합 테스트
  - 전체 테스트
  - README
  - 파이프라인 검증
tags:
  - Project
related:
  - "[[00_Seoul Station Real-Time Train Project|train-project]]"
  - "[[01_Docker_Setup_Postgresql_Setup]]"
  - "[[02_API_Producer]]"
  - "[[03_Kafka_Producer]]"
  - "[[04_Spark_Streaming]]"
  - "[[05_Superset_Dashboard]]"
  - "[[06_Airflow_Pipeline]]"
---

# 07_Integration_Test — 전체 통합 테스트 

## 이 단계의 목표

```
각 STEP 이 독립적으로는 동작했으니
전체 파이프라인을 처음부터 끝까지 한 번에 돌려보기
+ README 정리로 프로젝트 마무리
```

---

---

# ① 전체 파이프라인 체크리스트

## 컨테이너 상태

```bash
docker compose ps
```

```
확인 항목:
  train-kafka       running (healthy)  ✅
  train-postgres    running (healthy)  ✅
  train-spark-master running           ✅
  train-spark-worker running           ✅
  train-superset    running            ✅
  train-airflow     running            ✅
```

## 포트 접속 확인

|서비스|URL|확인 내용|
|---|---|---|
|Spark Web UI|http://localhost:8080|Workers: 1 표시|
|Airflow Web UI|http://localhost:8082|DAG 목록 표시|
|Superset|http://localhost:8088|대시보드 접속|
|PostgreSQL|localhost:5433|DataGrip 연결|

## Kafka 토픽 확인

```bash
docker exec -it train-kafka \
  /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 --list

# 아래 3개 있어야 함
# train-schedule
# train-realtime
# train-delay
```

---

---

# ② 실행 순서 최종 정리

```bash
# 1. 컨테이너 전체 실행
docker compose up -d

# 2. jar 파일 복사 (컨테이너 재시작 시 매번)
docker cp spark_drivers/. train-spark-master:/opt/spark/jars/

# 3. Kafka 토픽 확인 
docker exec -it train-kafka \
  /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 --list
  
# 토픽이 없으면 생성
docker exec -it train-kafka \
  /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create --topic train-schedule --partitions 1 --replication-factor 1

docker exec -it train-kafka \
  /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create --topic train-realtime --partitions 1 --replication-factor 1

docker exec -it train-kafka \
  /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create --topic train-delay --partitions 1 --replication-factor 1  
  


# 4. Producer 실행 (realtime 60초 폴링)
cd producer && python3 producer.py 

# 5. Spark Consumer 실행
docker exec -it train-spark-master \
  /opt/spark/bin/spark-submit \
  --master spark://spark-master:7077 \
  --jars /opt/spark/jars/spark-sql-kafka-0-10_2.12-3.5.0.jar,\
/opt/spark/jars/spark-token-provider-kafka-0-10_2.12-3.5.0.jar,\
/opt/spark/jars/kafka-clients-3.4.1.jar,\
/opt/spark/jars/commons-pool2-2.11.1.jar,\
/opt/spark/jars/postgresql-42.7.1.jar \
  /opt/spark/apps/consumer.py

# 6. Airflow DAG 활성화 (localhost:8082)
#    train_schedule_dag → 토글 ON
#    train_delay_dag    → 토글 ON
```

---

---

# ③ 데이터 흐름 검증

## PostgreSQL 데이터 확인

```bash
docker exec -it train-postgres psql -U train_user -d train_db
```

```sql
-- 운행계획 수집됐는지
SELECT COUNT(*), MAX(created_at) FROM train_schedule;

-- 실시간 데이터 쌓이는지 (60초마다 갱신)
SELECT COUNT(*), MAX(created_at) FROM train_realtime;

-- 지연 분석 데이터
SELECT COUNT(*), MAX(created_at) FROM train_delay;

-- 열차별 최신 상태 확인
SELECT DISTINCT ON (trn_no)
    trn_no, plan_dep, arvl_stn_nm, status, created_at
FROM train_realtime
ORDER BY trn_no, created_at DESC
LIMIT 10;
```

## Kafka 메시지 확인

```bash
# train-realtime 토픽 최신 메시지 확인
docker exec -it train-kafka \
  /opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic train-realtime \
  --from-beginning \
  --max-messages 5
```

## Airflow DAG 실행 확인

```bash
# 포그라운드로 스케줄러 실행해서 로그 직접 확인
docker exec -it train-airflow airflow scheduler
```

```
정상 실행 시 로그:
  INFO - [운행계획 DAG] 수집 완료 ✅ - 20260316
  INFO - [지연 정보 DAG] 수집 완료 ✅ - 20260315 기준
```

---

---

# ④ Superset 대시보드 최종 확인

```
http://localhost:8088

확인 항목:
  □ 어제 vs 오늘 운행횟수 Table — 숫자 표시
  □ 오늘 열차 운행횟수 Big Number — 숫자 표시
  □ 전날 기준 지연 횟수 Big Number — 숫자 표시
  □ 지금 곧출발하는 열차 Table — 데이터 or 메시지
  □ 오늘의 열차들은 어디로 Pie Chart — 차트 표시
  □ 실시간 기차 전광판 Table — 운행 상태 실시간 갱신
  □ 시간별 기차 이동 Line Chart — 그래프 표시
  □ 기차 운행 패턴 분석 Text — 내용 표시
  □ 노선별 지연 Bar Chart — 막대 표시

자동 새로고침: 30초 설정 확인
```

---
---

# ⑤ docker compose down 후 재시작 체크리스트

```
□ docker compose up -d
□ docker cp spark_drivers/. train-spark-master:/opt/spark/jars/
□ Kafka 토픽 확인 (없으면 재생성)
□ python3 producer/producer.py 재실행
□ spark-submit 재실행
□ Airflow Web UI → DAG 토글 ON 확인
□ producer_state.json 삭제 여부 확인
  (처음부터 다시 수집하려면 삭제)
```

---

✅ 프로젝트 완료 