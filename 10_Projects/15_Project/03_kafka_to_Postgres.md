---
aliases:
  - Postgres
  - SQL
  - kafka
tags:
  - Project
related:
  - "[[00_Project_HomePage]]"
  - "[[Python_Database_Connect]]"
  - "[[Kafka_Python_Consumer_Basic]]"
  - "[[SQL_DDL_Create]]"
  - "[[SQL_DML_CRUD]]"
  - "[[Airflow_Providers]]"
linked:
  - file:////Users/gong/gong_study_de/project/generator/fake_log_producer.py
  - file:////Users/gong/gong_study_de/project/streamlit_app/Dashboard.py
  - file:///Users/gong/gong_study_de/project/save_to_db.py
---
# kafka로 데이터를 주면, PostgreSQL 에 저장


## 라이브러리 설치

파이썬이 PostgreSQL 데이터베이스와 대화하려면 통역사(라이브러리)가 필요


```bash
# psycopg2-binary: 파이썬에서 포스트그레스 연결할 때 쓰는 표준 라이브러리
pip3 install psycopg2-binary
```

---
## 테이블 만들기 

데이터를 넣으려면 그에 맞는 **테이블(Table)** 이 DB에 먼저 있어야 합니다.
Faker에서 만든 데이터 구조와 똑같이 생성

1. Datagrip
2. docker postgres의 아이디 비번, port번호 입력 db까지 변경해줘야함! 
> postgre의 port번호는 기존의 로컬 DB 포트랑 똑같아서 5433으로 변경 
> 로컬DB 충돌 확인법 :터미널에 `lsof -i :5432`
3. TABLE 만들기
>왼쪽 `postgres@localhost` 더블클릭하거나,상단 `+` 버튼 → **Query Console** 클릭
>CREATE TABLE 코드만들기 
>실행 : `Cmd + Enter` 또는 상단 실행버튼클릭  
>생성확인 

```text
postgres@localhost
  └── postgres
       └── public
            └── Tables
                 └── user_logs  
```

```sql title="pake_log_producer.py에서 만들어둔 구조로 !"
DROP TABLE IF EXISTS user_logs;  
CREATE TABLE user_logs(  
    event_id uuid PRIMARY KEY,  
    order_id uuid,  
    customer_id uuid,  
    product_id uuid,  
  
    order_status VARCHAR(50),  
    payment_type VARCHAR(50),  
    price INT,  
    quantity INT,  
    total_amount INT,  
    category VARCHAR(50),  
    product_name TEXT,  
  
    timestamp TIMESTAMP,  
    customer_name VARCHAR(100),  
    customer_city VARCHAR(100),  
    customer_state VARCHAR(100)  
)
```

> `fake_log_producer.py`가 생성하는 키(Key) 이름들과 순서가 맞아야 합니다!
> **`uuid` 타입**: 전 세계에서 유일한 ID를 보장하며, 검색 속도와 용량 효율을 극대화합니다.
> **`TEXT` 타입**: 상품명 등 길이를 예측할 수 없는 문자열에 대해 에러 없이 유연하게 대응

---
## kafka 데이터를 DB에넣는 `save_to_db`

```python title='save_to_db.py'
from kafka import KafkaConsumer
import json
import psycopg2
import os
from dotenv import load_dotenv

# 1. 환경변수 로드: 보안의 시작! 소스코드에 비번을 직접 적지 않는 습관입니다.
load_dotenv()

# 2. DB 접속 정보: **를 붙여서 psycopg2.connect()에 한 방에 던져줄 가방입니다.
DB_CONFIG = {
    "host": os.getenv("DB_HOST"),
    "port": os.getenv("DB_PORT"),
    "database": os.getenv("DB_NAME"),
    "user": os.getenv("DB_USER"),
    "password": os.getenv("DB_PASSWORD")
}

# 3. 카프카 소비자 설정: 'user-log' 토픽에서 실시간으로 데이터를 빨아들이는 빨대 역할을 합니다.
consumer = KafkaConsumer(
    'user-log',
    bootstrap_servers=['localhost:29092'],
    auto_offset_reset='latest',  # 가장 최신 데이터부터 읽기 시작
    value_deserializer=lambda x: json.loads(x.decode('utf-8')) # bytes -> JSON(딕셔너리) 변환
)

def insert_data(conn, cur, data):
    # 4. SQL 뼈대: %s는 나중에 파이썬 변수가 들어갈 '예약석'입니다. 따옴표를 안 씁니다!
    sql = """
        INSERT INTO user_logs(
            event_id, order_id, customer_id, product_id,
            order_status, payment_type, price, quantity, total_amount,
            category, product_name,
            timestamp,
            customer_name, customer_city, customer_state
        )
        VALUES(
            %s, %s, %s, %s,
            %s, %s, %s, %s, %s,
            %s, %s,
            NOW(),  -- 파이썬에서 안 주고 DB가 직접 시간을 찍게 함 (중요 ⭐)
            %s, %s, %s
        )
    """
    
    # 5. 데이터 매핑: SQL의 %s 개수와 이 튜플의 개수는 반드시 1:1로 맞아야 합니다!
    values = (
        data.get("event_id"),
        data.get("order_id"),
        data.get("customer_id"),
        data.get("product_id"),
        data.get("order_status"),
        data.get("payment_type"),
        data.get("price"),
        data.get("quantity"),
        data.get("total_amount"),
        data.get("category"),
        data.get("product_name"),
        # NOW() 자리는 여기서 생략!
        data.get("customer_name"),
        data.get("customer_city"),
        data.get("customer_state")
    )
    
    # 6. 실행 및 확정: execute로 주문 넣고, commit으로 결제 도장을 찍습니다.
    cur.execute(sql, values)
    conn.commit() 
    
def connect_sql_python():
    print("[start] kafka데이터 -> PostgreSQL저장 시작!")
    
    conn = None # 7. 안전장치: 에러가 나도 finally에서 close()를 호출할 수 있게 미리 선언
    
    try:
        # 8. 연결 및 커서: 식당 입장(conn) 후 주문받을 웨이터(cur)를 부릅니다.
        conn = psycopg2.connect(**DB_CONFIG)
        cur = conn.cursor()
        print("DB 연결되었다!")
        
        # 9. 무한 루프: 카프카에 새로운 데이터가 들어올 때까지 계속 기다리며 처리합니다.
        for message in consumer:
            row = message.value
            insert_data(conn, cur, row)
            print(f"저장 성공: {row.get('order_id','Unknown')} - {row.get('category','Unknown')}")
            
    except Exception as e:
        # 10. 롤백: 작업 중 에러 나면 지금까지 했던 걸 다 취소하고 깨끗하게 되돌립니다.
        print(f"Error!! {e}")
        if conn:
            conn.rollback()
    
    except KeyboardInterrupt:
        # 터미널에서 Ctrl + C를 눌러 중단했을 때를 위한 매너 있는 종료
        print("Stopping by User..")
        
    finally:
        # 11. 뒷정리: 연결을 끊지 않으면 DB 서버의 자원이 낭비되므로 무조건 닫아줍니다.
        if conn:
            cur.close()
            conn.close()
            print("DB와의 연결이 끊어졌습니다.")
            
if __name__ == "__main__":
    connect_sql_python()
```


## 실행해보기

**Terminal 1 (데이터 생성):**

```bash
python3 generator/fake_log_producer.py
```

**Terminal 2 (DB 적재):** (방금 만든 거!)

```bash
python3 save_to_db.py
```

 **Terminal 3** 대시보드 

```bash
python3 -m streamlit run streamlit_app/Dashboard.py
```

---
## DataGrip 확인 

1. **테이블 더블 클릭**: 생성한 `user_logs` 테이블을 더블 클릭
2. `user_logs` 테이블 위에서 마우스 우클릭 -> **[Query Console]** 클릭
3. 분석 쿼리를 타이핑
4. **실행**: 쿼리 위에 커서를 두고 `Cmd + Enter` 실행! 

```sql
SELECT  category,customer_name,customer_city,customer_state,price  
FROM user_logs  
ORDER BY customer_name  
  
SELECT category,COUNT(category) as order_count,SUM(total_amount)as total_sales  
FROM user_logs  
GROUP BY category  
ORDER BY total_sales DESC
```

> 성공!!!!

