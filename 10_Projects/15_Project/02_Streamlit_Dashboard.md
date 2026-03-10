---
aliases:
  - Stramlit
tags:
  - Project
related:
  - "[[00_Project_HomePage]]"
  - "[[Kafka_Python_Consumer]]"
  - "[[Python_JSON]]"
  - "[[Python_DateTime]]"
  - "[[Streamlit_Layout]]"
  - "[[Streamlit_Widgets]]"
  - "[[Streamlit_Concept]]"
  - "[[Streamlit_Session_State]]"
  - "[[Pandas_Datatypes_Filtering]]"
  - "[[Python_Pathlib]]"
  - "[[Pandas_Groupby]]"
  - "[[Pandas_Apply_Map]]"
  - "[[Streamlit_Data_Display]]"
linked:
  - file:////Users/gong/gong_study_de/project/streamlit_app/Dashboard.py
  - file:////Users/gong/gong_study_de/project/generator/fake_log_producer.py
---
# Streamlit Dashboard 만들어서 실시간 모니터링하기 (Consumer)

Kafka에서 실시간으로 넘어오는 데이터를 받아 시각화하는 대시보드입니다.

## 실행 명령어

터미널에서 프로젝트 루트 경로로 이동한 뒤 실행합니다.

```bash
python3 -m streamlit run streamlit_app/Dashboard.py
```

----
## 전체 코드 (`Dashboard.py`)

```python title='Dashboard.py'
import streamlit as st
from kafka import KafkaConsumer
from kafka.errors import NoBrokersAvailable
import json
import pandas as pd
import time
from pathlib import Path  

# [설정] 페이지 기본 세팅
st.set_page_config(
    page_title="Data PipeLine",
    page_icon="✅",
    layout="wide"
)

st.title("Kafka Dashboard")
st.divider()


# --- [Layout 구성] 상단 4개 컬럼 ---
m1, m2, m3, m4 = st.columns(4)
with m1:
    st.info("총 주문건수")
    data_total_count = st.empty()
with m2:
    st.info("최근 주문금액")
    data_last_value = st.empty()
with m3:
    st.info("최근 카테고리")
    data_category_status = st.empty()
with m4:
    data_connect_status = st.empty()  # 연결 상태 표시

st.divider()

# --- [Layout 구성] 중단: 카테고리별 통계 ---
st.subheader("카테고리별 실시간 통계")
category_table_slot = st.empty()

st.markdown("### Streaming Data & Logs")

# --- [Layout 구성] 하단: 차트와 로그 ---
col_chart, col_log = st.columns([2, 1])
with col_chart:
    st.caption("Chart")
    chart_placeholder = st.empty()
with col_log:
    st.caption("Data Log(실시간)")
    log_placeholder = st.empty()


# --- [Kafka 연결] ---
try:
    consumer = KafkaConsumer(
        'user-log',
        bootstrap_servers=['localhost:29092'], # Producer와 동일한 포트
        auto_offset_reset='latest',
        value_deserializer=lambda x: json.loads(x.decode('utf-8')),
        consumer_timeout_ms=1000
    )
    data_connect_status.success("Kafka랑 연결되었다")
except NoBrokersAvailable:
    st.error("🚫 Broker Unavailable. Kafka가 꺼져 있나요?")
    st.stop()
except Exception as e:
    st.error(f"Error! {e}")
    st.stop()
    
# --- [데이터 수집 준비] ---
if "user_logs" not in st.session_state:
    st.session_state.user_logs = []
    
total_count = 0

# --- [제어 기능] 종료 신호 보내기 (Pathlib) ---
if st.sidebar.button("모니터링 그만하고 싶다면 클릭"):
    # pathlib로 파일 생성 + 쓰기 + 닫기 한 번에 처리
    Path(".stop_signal").write_text("stop", encoding="utf-8")
    
    st.toast("종료버튼 클릭했습니다.")
    # st.success("producer에게 종료신호를 보냅니다.")
    st.stop()


# --- [메인 루프] 실시간 데이터 처리 ---
for message in consumer:
    row = message.value
    # data_connect_status.success(f"데이터 수신 중! 마지막 ID: {row['order_id'][:8]}...")
    
    #  데이터 버퍼링
    st.session_state.user_logs.append(row)
    total_count += 1
    
    # 버퍼 관리 (최신 100개 유지)
    if len(st.session_state.user_logs) > 100:
        st.session_state.user_logs.pop(0)
        
    df = pd.DataFrame(st.session_state.user_logs)
    
    #  상단 지표 업데이트
    data_total_count.metric(label="Total Message", value=total_count)
    
    # 금액 포맷팅 (정수 변환 + 천 단위 콤마)
    data_last_value.metric(label="마지막 금액", value=f"{int(row['total_amount']):,}원")
    data_category_status.metric(label="최근 카테고리", value=row['category'])
    data_connect_status.success(f"연결됨: {row['order_id'][:8]}...")
    
    
    #  카테고리별 요약 (Pandas Groupby)
    df_summary = df.groupby('category').agg(
        건수=('order_id', 'count'),
        매출액=('total_amount', 'sum')
    ).reset_index()
    
    # 보기 좋게 포맷팅 (옵션)
    df_summary['매출액'] = df_summary['매출액'].apply(lambda x: f"{int(x):,}원")
    
    category_table_slot.dataframe(df_summary, use_container_width=True,hide_index=True)
    
    
    #  차트 업데이트 (숫자 컬럼 자동 감지)
    numeric_cols = df.select_dtypes(include=['number']).columns
    if not df.empty and len(numeric_cols) > 0:
        chart_placeholder.line_chart(df[[numeric_cols[0]]])
        
    # 6. 로그 테이블 업데이트 (최신순 정렬)
    log_placeholder.dataframe(df.tail(10).iloc[::-1], use_container_width=True)
    
    time.sleep(0.05)
```


---
## Producer 코드 업데이트 (`fake_log_producer.py`)

대시보드 종료 버튼과 연동하기 위해 **시작 전 청소(unlink)**와 **실시간 감지(exists)** 로직을 추가

```python title='fake_log_producer.py 업데이트'
from pathlib import Path  # [필수] 맨 위에 추가
# ... (기존 import) ...

if __name__ == "__main__":
    # [1] 시작 전 청소: 혹시 남아있을지 모를 종료 신호 파일 제거 (중요!)
    # missing_ok=True: 파일이 없어도 에러 내지 말고 넘어가라는 뜻
    Path(".stop_signal").unlink(missing_ok=True)

    print(f"[Start] sending {TOPIC_NAME} topic!")
    producer = None
    
    # Kafka 연결 시도 루프
    for i in range(10):
        try:
            # ... (연결 코드) ...
            break
        except NoBrokersAvailable:
            # ... (재시도 코드) ...
            time.sleep(3)
            
    # ... (중략) ...
    
    # [2] 메인 전송 루프
    try:
        while True:
            # --- [종료 신호 감지 로직] ---
            stop_file = Path(".stop_signal")
            
            if stop_file.exists():
                print("🛑 Dashboard에서 종료버튼 클릭! Producer를 멈춥니다.")
                stop_file.unlink()  # 신호 파일 삭제 (다음 실행을 위해 청소)
                break 
            # ---------------------------

            log = generate_user_data()
            producer.send(TOPIC_NAME, value=log)
            # ... (기존 출력 및 sleep) ...
            
    except KeyboardInterrupt:
        print("Stop..!")
        producer.close()
```

---
## 실행 명령어 (터미널 2개 필요!)

대시보드가 작동하려면 **데이터를 보내주는 방송국(Producer)** 과 **데이터를 받는 TV(Dashboard)** 가 동시에 켜져 있어야 합니다.

### ① 터미널 A: 데이터 생성기 실행 (Producer)

먼저 가짜 데이터를 만들어서 Kafka로 쏘는 프로그램을 실행합니다.

```bash
python3 generator/fake_log_producer.py
```

### ② 터미널 B: 대시보드 실행 (Dashboard)

새 터미널을 열고, 데이터를 받아서 화면에 그려주는 대시보드를 실행합니다.

```bash
python3 -m streamlit run streamlit_app/Dashboard.py
```
