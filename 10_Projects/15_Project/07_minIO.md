---
aliases:
  - minIO
tags:
  - Project
related:
  - "[[00_Project_HomePage]]"
  - "[[Kafka_Python_Consumer]]"
  - "[[MinIO_Integration]]"
  - "[[Pandas_Read_Write]]"
  - "[[MinIO_Concept]]"
  - "[[Kafka_to_MinIO_DataLake]]"
  - "[[Buffering_vs_Batching]]"
  - "[[Python_DateTime]]"
  - "[[Pandas_DataStructures]]"
  - "[[MinIO_Docker_Setup]]"
  - "[[04_Spark_Batch]]"
---
# Kafka to MInIO -Kafka ->MinIO Parquet저장 ->Dashboard 확인

## 1. MinIO에 '버킷(Bucket)' 생성 raw-data

[localhost:9001](http://localhost:9001/browser)

## Kafka_to_minio.py

```python
from kafka import KafkaConsumer
import pandas as pd
import json
from datetime import datetime
import os
from dotenv import load_dotenv

# ==========================================
# 1. 환경 변수 로드 (.env 파일에서 보안 정보 가져오기)
# ==========================================
load_dotenv()

# S3(MinIO) 접속을 위한 인증 정보 및 주소 셋팅
MINIO_ENDPOINT=os.getenv('MINIO_ENDPOINT')     # MinIO 서버 주소 (예: http://localhost:9000)
MINIO_ACCESS_KEY=os.getenv('MINIO_ACCESS_KEY') # 아이디
MINIO_SECRET_KEY=os.getenv('MINIO_SECRET_KEY') # 비밀번호
BUCKET_NAME=os.getenv('MINIO_BUCKET_NAME')     # 저장할 버킷(폴더) 이름

# Pandas가 S3(MinIO)와 통신할 때 사용할 '출입증 및 통역사' 역할의 딕셔너리
storage_options={
    'key': MINIO_ACCESS_KEY,
    'secret': MINIO_SECRET_KEY,
    'client_kwargs': {"endpoint_url": MINIO_ENDPOINT} # 실제 AWS S3가 아닌 우리만의 MinIO 주소로 향하게 하는 핵심 옵션!
}

# ==========================================
# 2. Kafka Consumer (데이터 수신 대기조) 셋팅
# ==========================================
consumer=KafkaConsumer(
    'user-log',                             # 구독할 Kafka 토픽 이름 
    bootstrap_servers=['localhost:29092'],  # Kafka 브로커 주소
    auto_offset_reset='latest',             # 컨슈머가 켜진 순간부터 들어오는 '최신' 데이터만 읽음 
    value_deserializer=lambda x:json.loads(x.decode('utf-8')) # 바이트 단위로 들어온 데이터를 Python 딕셔너리로 변환(디코딩)
)

print("🚀 Data Lake(MinIO) 적재 파이프라인 실행 중!!")
print(f"📦 설정된 Storage Options: {storage_options}")

# ==========================================
# 3. 마이크로 배치(Micro-batch) 설정
# ==========================================
# 한 건씩 저장하면 네트워크 부하가 심하고 'Small File Problem'이 발생하므로,
# 메모리(바구니)에 임시로 모아둘 공간과 한계치를 설정합니다.
buffer = []
BATCH_SIZE = 10 # 10건이 모일 때마다 1개의 파일로 압축해서 저장! (실무에서는 1000건, 10000건 단위로 설정함)

# ==========================================
# 4. 실시간 데이터 스트리밍 및 Data Lake 저장 로직
# ==========================================
for message in consumer:
    # 4-1. Kafka에서 꺼낸 데이터를 바구니(buffer)에 담기
    buffer.append(message.value)
    print(f"📦 바구니에 담는 중... ({len(buffer)}/{BATCH_SIZE})")
    
    # 4-2. 바구니가 가득 차면 (배치 사이즈 도달) 창고(MinIO)로 배송 시작!
    if len(buffer) >= BATCH_SIZE:
        # 모아둔 딕셔너리 리스트를 표(DataFrame) 형태로 변환
        df = pd.DataFrame(buffer)
        
        # 파일명 생성: 현재 시간을 기반으로 유니크한 이름 부여 (예: 20260222_123045_raw.parquet)
        now = datetime.now().strftime("%Y%m%d_%H%M%S")
        file_path = f"s3://{BUCKET_NAME}/{now}_raw.parquet"
        
        try:
            # 4-3. DataFrame을 Parquet 포맷으로 압축하여 MinIO에 업로드
            df.to_parquet(
                file_path,
                engine="pyarrow",      # Parquet 변환 엔진
                compression="snappy",  # 빅데이터 표준 압축 방식 (가볍고 빠름)
                index=False,           # DataFrame의 숫자 인덱스는 불필요하므로 제외
                storage_options=storage_options # 아까 만든 출입증 제시!
            )
            print(f"✅ 저장 완료: {file_path}")
            
        except Exception as e:
            # 주소 오류, 권한 오류 등 저장 실패 시 에러 로그 출력
            print(f"❌ 저장 실패: {e}")
            
        finally:
            #  다음 작업을 위해 바구니를 깨끗하게 비움
            buffer.clear()
```

## 실행 

### minio 실행

```bash
python3 kafka_to_minio.py
```

### producer 실행

```bash
python3 generator/fake_log_producer.py
```

>저장완료:s3://raw-data/20260222_194716_raw.parquet 뜨는 것을 확인할수 있다
>http://localhost:9001/browser/raw-data 여기에서 들어오는 것을 확인할 수 있다!


## 생성된 parquet로 daily_report 수정해보기! 

### SparkSession 에 S3A설정을 추가하기

```python
# ==========================================
# SparkSession 생성 및 Data Lake(MinIO) 연동 설정
# ==========================================
spark = (
    SparkSession.builder
    .appName("MinIO_DataLake_Report")

    # 1. 인증 설정 (Authentication)
    .config("spark.hadoop.fs.s3a.endpoint", os.getenv("MINIO_ENDPOINT"))
    .config("spark.hadoop.fs.s3a.access.key", os.getenv("MINIO_ACCESS_KEY"))
    .config("spark.hadoop.fs.s3a.secret.key", os.getenv("MINIO_SECRET_KEY"))

    # 2. S3A 동작 및 보안 설정 (Behavior & Security)
    # 실제 AWS S3가 아닌 MinIO를 사용하기 위한 필수 우회 설정들입니다.
    .config("spark.hadoop.fs.s3a.path.style.access", "true")  # MinIO 호환성을 위한 Path-style access 활성화
    .config("spark.hadoop.fs.s3a.impl", "org.apache.hadoop.fs.s3a.S3AFileSystem")
    .config("spark.hadoop.fs.s3a.aws.credentials.provider", "org.apache.hadoop.fs.s3a.SimpleAWSCredentialsProvider")
    .config("spark.hadoop.fs.s3a.connection.ssl.enabled", "false")  # 내부망(로컬) 통신을 위해 HTTPS(SSL) 비활성화

    # 3. 네트워크 타임아웃 및 재시도 설정 (Network Resiliency)
    # Spark가 네트워크를 못 찾고 무한 대기(Hang)에 빠지는 것을 방지하는 안전장치입니다.
    .config("spark.hadoop.fs.s3a.connection.timeout", "10000")           # 연결 유지 타임아웃 (10초)
    .config("spark.hadoop.fs.s3a.connection.establish.timeout", "5000")  # 초기 연결 타임아웃 (5초)
    .config("spark.hadoop.fs.s3a.attempts.maximum", "3")                 # 최대 재시도 횟수 (3회)

    # 4. 성능 최적화 (Performance Tuning)
    # 대용량 데이터를 Parquet으로 읽고 쓸 때 속도를 높여주는 옵션입니다.
    .config("spark.hadoop.fs.s3a.multipart.size", "104857600")  # 멀티파트 업로드 청크 사이즈 (100MB)
    .config("spark.hadoop.fs.s3a.fast.upload", "true")          # 메모리/디스크 버퍼를 활용한 빠른 업로드 활성화

    .getOrCreate()
)
```

> 잠깐! 여기서 실행하려고 할대는 MINIO_ENDPOINT가 localhost가 아니라 MINIO_ENDPOINT=http://minio:9000 이렇게 수정해놔야함!

## Docker 네트워크 문제 

```bash
docker exec -it spark-master /opt/spark/bin/spark-submit \
  --master spark://spark-master:7077 \
  --jars /opt/spark/jars/postgresql-42.7.1.jar \
  --packages org.apache.hadoop:hadoop-aws:3.3.4,com.amazonaws:aws-java-sdk-bundle:1.12.262 \
  /opt/spark/apps/daily_report.py
```

실행하려고 했는데 계속 실행이 안되고 멈춰있는 현상 발생 > minio에 network를 추가 안해줘서 통신불가상태여서 network추가 

## streamlit Dashboard

```python
# ==========================================================
# 📊 Spark Batch Analysis 섹션 (람다 아키텍처의 Batch Layer 표현)
# ==========================================================
st.subheader("Spark Batch Analysis (Batch Layer)")

# 탭을 사용하여 두 가지 데이터 처리 방식의 결과를 직관적으로 비교
tab1, tab2 = st.tabs(["🗄️ PostgreSQL 기반", "📦 Data Lake(MinIO) 기반"])

# --- [Tab 1] 운영 DB 직접 분석 섹션 ---
with tab1:
    st.caption("PostgreSQL의 원본 데이터를 직접 분석한 결과 (현재 상태 확인용)")
    db_report_col1, db_report_col2 = st.columns(2)
    
    try:
        # DB의 report_category_sales 테이블에서 집계 데이터 로드
        df_db = pd.read_sql("SELECT * FROM report_category_sales", engine)
        
        with db_report_col1:
            total_db_sales = df_db['total_sales'].sum()
            # 전체 매출액을 강조 지표(Metric)로 표시
            st.metric("DB 기반 총 매출액", f"{int(total_db_sales):,}원")
            st.dataframe(df_db, use_container_width=True, hide_index=True)
            
        with db_report_col2:
            # Plotly를 활용한 카테고리별 매출 막대 그래프
            fig = px.bar(df_db, x="category", y="total_sales", title="카테고리별 매출 (DB)",
                         color="category", text_auto=',.0f')
            st.plotly_chart(fig, use_container_width=True, key="db_chart")
            
    except Exception as e:
        # 데이터가 아직 없거나 DB 연결 문제 발생 시 경고 표시
        st.warning(f"DB 기반 리포트 대기 중... ({e})")

# --- [Tab 2] Data Lake(MinIO) + Spark 분석 섹션 ---
with tab2:
    st.caption("MinIO에 저장된 Parquet 파일을 Spark로 대용량 분석한 결과 (확장성 강조)")
    minio_report_col1, minio_report_col2 = st.columns(2)
    
    try:
        # Spark Job이 MinIO 데이터를 가공하여 DB에 저장해둔 결과 테이블 로드
        df_minio = pd.read_sql("SELECT * FROM report_category_sales_minio", engine)
        
        with minio_report_col1:
            total_minio_sales = df_minio['total_sales'].sum()
            st.metric("Data Lake 기반 총 매출액", f"{int(total_minio_sales):,}원")
            st.dataframe(df_minio, use_container_width=True, hide_index=True)
            
        with minio_report_col2:
            # DB 결과와 구분되도록 파스텔톤 컬러(Pastel) 적용 및 시각화
            fig_minio = px.bar(df_minio, x="category", y="total_sales", title="카테고리별 매출 (MinIO)",
                               color="category", text_auto=',.0f', color_discrete_sequence=px.colors.qualitative.Pastel)
            st.plotly_chart(fig_minio, use_container_width=True, key="minio_chart")
            
    except Exception as e:
        # Spark Batch Job이 실행되지 않았을 경우 안내 메시지 출력
        st.warning(f"Data Lake 리포트를 생성해주세요! (Spark Job 실행 필요)")

st.divider() # 섹션 구분을 위한 가로선
```

## 확인 

![[스크린샷 2026-02-22 오후 8.42.29.png|700x400]]