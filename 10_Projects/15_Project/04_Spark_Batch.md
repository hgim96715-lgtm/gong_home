---
aliases:
tags:
  - Project
related:
  - "[[Spark_JDBC_PostgreSQL]]"
  - "[[PySpark_Session_Context]]"
  - "[[Spark_Session_Deep_Dive]]"
  - "[[Python_Database_Connect]]"
  - "[[Spark_DataFrame]]"
  - "[[Spark_Data_IO]]"
  - "[[Docker_Host_vs_Internal_Network]]"
  - "[[Python_Database_Connect]]"
  - "[[00_Project_HomePage]]"
linked:
  - file:////Users/gong/gong_study_de/project/streamlit_app/Dashboard.py
  - file:////Users/gong/gong_study_de/project/spark_app/daily_report.py
---
# PostgreSQL 데이터 분석 >Spark로 DB 카테고리별 매출 집계 후 Dashboard 연동

## 흐름

```scss
PostgreSQL (user_logs)
        ↓  spark.read.jdbc()
    Spark 집계
    (카테고리별 매출 합계 / 주문 건수)
        ↓  df.write.jdbc()
PostgreSQL (report_category_sales)
        ↓  pd.read_sql()
  Streamlit Dashboard
```

>Kafka로 쌓인 원본 로그(`user_logs`)를 Spark로 읽어 카테고리별로 집계하고, 결과를 새 테이블(`report_category_sales`)에 저장해 대시보드에 띄우기

## 네트워크 구조 (중요)

이것때문에 계속 실패를 거듭했음! 중요함 !!!

>[[Docker_Host_vs_Internal_Network]] 정리한 것 한번 더 확인!!!

|실행 위치|호스트명|이유|
|---|---|---|
|Spark (도커 내부)|`postgres`|도커 내부 컨테이너명으로 통신|
|Streamlit (로컬 Mac)|`localhost`|도커 밖에서는 포트포워딩으로 접근|

---
## Daily_report.py (Spark 배치 )

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, sum, count, desc
from dotenv import load_dotenv
import os

# 1. 환경 변수 로드: .env 파일에 정의된 DB 접속 정보를 시스템 환경 변수로 불러옵니다.
load_dotenv()

def daily_report():
    # 2. Spark 세션 생성: Spark 앱의 이름(DailyReport)을 설정하고 시동을 겁니다.
    spark = (
        SparkSession
        .builder
        .appName("DailyReport")
        .getOrCreate()
    )
    
    print("\n" + "="*50)
    print(">>> Spark: PostgreSQL 연결 시도 중...")
    
    # 3. JDBC 접속 설정
    # [중요] 도커 내부 네트워크망을 통해 'postgres' 컨테이너의 '5432' 포트로 직접 통신합니다.
    jdbc_url = "jdbc:postgresql://postgres:5432/airflow"
    
    # [안전장치] os.getenv에 기본값(default)을 설정하여 Java의 NullPointerException을 방지합니다.
    db_properties = {
        "user": os.getenv("DB_USER", "airflow"),
        "password": os.getenv("DB_PASSWORD", "airflow"),
        "driver": "org.postgresql.Driver"  # PostgreSQL 통역사(Driver) 지정
    }
    
    try:
        # 4. 데이터 읽기 (Read): 원본 데이터인 'user_logs' 테이블을 Spark로 불러옵니다.
        df = spark.read.jdbc(url=jdbc_url, table="user_logs", properties=db_properties)
        
        print(f">>> 데이터 로드 성공! (총 {df.count()}건)")
        
        # 5. 데이터 가공 (Transformation): 카테고리별 매출 합계와 주문 건수를 집계합니다.
        report_df = (
            df
            .groupBy("category")
            .agg(
                count("order_id").alias("total_orders"),    # 주문 개수 세기
                sum("total_amount").alias("total_sales")    # 매출 합계 더하기
            )
            .orderBy(col("total_sales").desc())             # 매출 높은 순으로 정렬
        )
        
        # 6. 결과 출력 (Action): 가공된 리포트의 상위 내용을 화면에 표 형태로 보여줍니다.
        print("\n[  오늘의 카테고리별 매출 Report ]")
        report_df.show()
        
        # 7. 결과 저장 (Write): 가공된 리포트를 DB의 새로운 테이블에 저장합니다.
        # mode="overwrite": 기존에 테이블이 있다면 싹 지우고 새로 작성하여 리포트를 최신화합니다. 굳이 지울필요 없음!
        print(">>> DB에 Report 저장 중...")
        report_df.write.jdbc(
            url=jdbc_url,
            table="report_category_sales", # 저장될 테이블명 (없으면 자동 생성)
            mode="overwrite",               # 덮어쓰기 모드
            properties=db_properties
        )
        print(">>> 저장 완료!!!")
        
    except Exception as e:
        # 에러 발생 시 로그를 남겨 디버깅을 돕습니다.
        print(f"\n Error ! {e}")
        
    finally:
        # 8. 자원 반납: 작업이 완료되면 Spark를 안전하게 종료하여 메모리를 해제합니다.
        spark.stop()
    
if __name__ == "__main__":
    daily_report()
```

---
## Dashboard 연동 (Streamlit)

Spark가 저장한 `report_category_sales` 테이블을 읽어 화면에 표시한다. Streamlit은 로컬에서 실행되므로 호스트를 `localhost`로 접근!!!

```python
import plotly.express as px
import os
from dotenv import load_dotenv
from sqlalchemy import create_engine

# 1. DB 연결 설정 
db_url = f"postgresql://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}@{os.getenv('DB_HOST')}:{os.getenv('DB_PORT')}/{os.getenv('DB_NAME')}"
engine = create_engine(db_url)

st.divider()
st.subheader("📊 Spark Batch Analysis")

# 2. 레이아웃 설정
db_report_col1, db_report_col2 = st.columns(2)

with db_report_col1:
    st.caption("📂 카테고리별 누적 집계")
    data_db_sales = st.empty()  # 매출 메트릭이 들어갈 자리
    db_table_slot = st.empty()  # 데이터 테이블이 들어갈 자리
    
with db_report_col2:
    st.caption("📈 매출 비중 시각화")
    db_chart_slot = st.empty()  # Plotly 차트가 들어갈 자리

# 3. 데이터 로드 및 시각화 로직
try:
    # DB에서 Spark가 저장한 리포트 테이블 읽기
    df_db = pd.read_sql("SELECT * FROM report_category_sales", engine)
    
    if not df_db.empty:
        # (1) 메트릭 업데이트: 총 매출액 계산
        total_db_sales = df_db['total_sales'].sum()
        data_db_sales.metric("💰 총 매출액", f"{int(total_db_sales):,}원")
        
        # (2) 테이블 업데이트
        db_table_slot.dataframe(df_db, use_container_width=True, hide_index=True)
        
        # (3) Plotly 차트 생성 및 업데이트
        fig = px.bar(
            df_db,
            x="category",
            y="total_sales",
            title="카테고리별 매출 현황",
            color="category",
            text="total_sales",
            template="plotly_white"  # 깔끔한 테마 적용
        )
        
        # 차트 디테일 설정 (천단위 콤마, 레이아웃 등)
        fig.update_traces(texttemplate='%{text:,}원', textposition='outside')
        fig.update_layout(
            showlegend=False, 
            xaxis_title="카테고리", 
            yaxis_title="매출액(원)",
            margin=dict(l=20, r=20, t=50, b=20)
        )
        
        # 슬롯에 차트 쏘기!
        db_chart_slot.plotly_chart(fig, use_container_width=True)
    else:
        db_table_slot.info("조회할 데이터가 없습니다.")

except Exception as e:
    db_table_slot.warning(f"⚠️ DB 연결 혹은 테이블 확인 필요: {e}")
```

>**💡 중요!!!!!!:** `st.empty()` 슬롯을 루프 밖에서 먼저 선언하고, 데이터 렌더링도 루프 밖에서 해야 Kafka 메시지 없어도 화면에 보인다.
>**❌ 실수:** DB 렌더링 코드를 `for message in consumer:` 루프 **안에** 넣으면 Kafka 메시지가 들어오는 동안에만 실행되기 때문에, 메시지가 없는 순간엔 절대 화면에 나타나지 않는다. Kafka 스트리밍과 무관하게 항상 보여야 하는 데이터는 반드시 루프 **밖**에서 렌더링할 것.


---
## 실행 

**터미널 1** 데이터 생성 

```bash
python3 generator/fake_log_producer.py
```

**터미널2** kafka데이터 ->postegreSQL저장

```bash
python3 save_to_db.py
```

**터미널 3** 대시보드 확인

```bash
python3 -m streamlit run streamlit_app/Dashboard.py
```

**터미널4** 다 멈추고 PostgreSQL에 넣은 데이터 카테고리별 합계 구하기 (producer 멈춘 뒤)

```bash
 docker exec -it spark-master /opt/spark/bin/spark-submit \
  --master spark://spark-master:7077 \
  --jars /opt/spark/jars/postgresql-42.7.1.jar \
  /opt/spark/apps/daily_report.py
```