---
aliases:
  - foreach
  - collect
  - 스파크반복문
  - OOM
  - Driver_vs_Executor
tags:
  - Spark
related:
  - "[[Spark_Architecture]]"
  - "[[Transformations_vs_Actions]]"
  - "[[Python_Control_Flow]]"
  - "[[00_Apache_Spark_HomePage]]"
---
## 개념 한 줄 요약 

**"데이터를 내 책상으로 가져올래(`collect`), 아니면 일꾼한테 가서 처리하라고 시킬래(`foreach`)?"**

| 메서드 | 실행 위치 | 비유 | 특징 |
| :--- | :--- | :--- | :--- |
| **`collect()`** | **Driver** (내 컴퓨터) | **"전교생 운동장으로 집합!"** | 모든 데이터를 가져오므로 메모리 터질 위험 있음 (OOM). |
| **`foreach()`** | **Executor** (일꾼 노드) | **"각 반 교실에서 방송 듣고 행동해!"** | 일꾼들이 각자 자리에서 병렬로 처리함 (빠름, 안전). |

---
##  죽음의 반복문: `collect()` 

**"절대 for문 돌리지 마세요!"** (작은 데이터 확인용으로만 사용)

```python
# ❌ [나쁜 예] 1TB 데이터를 내 노트북으로 가져와서 반복문 돌리기
# 결과: Driver 메모리 초과로 프로그램 멈춤 (OOM)
results = rdd.collect() 
for item in results:
    print(item)
```

**언제 쓰는가?**

1. `word_count` 결과처럼 데이터가 확 줄어들어서(수천 건 이내) 한 눈에 보고 싶을 때.
2. 디버깅용으로 `take(5)` 한 소량의 데이터를 확인할 때.

---
## 진정한 분산 반복문: `foreach()` 

데이터가 아무리 커도 터지지 않는 **Action(실행)** 함수입니다.

### 문법 (Syntax)

가장 기본적인 사용법입니다. 함수를 정의하고 던져주면 됩니다.

- 문법 : `{python}rdd.foreach(함수이름)`

```python
# 1. 처리할 로직(함수) 정의
def print_data(x):
    print(f"데이터 처리: {x}")

# 2. 적용 (함수 이름만 넣기)
rdd.foreach(print_data)

# 3. (또는) 람다식으로 한 줄 컷
rdd.foreach(lambda x: print(f"데이터 처리: {x}"))
```

---
## 왜 `print` 해도 화면에 안 나와요? 

Jupyter에서 `rdd.foreach(print)`를 실행하면 에러도 안 나는데 **화면에 아무것도 출력되지 않습니다.** 
이 현상을 이해하는 게 스파크의 핵심입니다.

### "리모컨과 TV" 비유

- **Jupyter (Driver):** 당신은 손에 **리모컨**을 쥐고 있습니다.
- **Worker (Executor):** 저 멀리 옆방에 **TV(일꾼)** 가 있습니다.
- **상황:** 리모컨으로 "전원 켜!"(print) 버튼을 눌렀습니다.
- **결과:** 옆방에 있는 **TV가 켜졌습니다.** 하지만 **당신 눈앞(Jupyter 화면)에는 아무 변화가 없습니다.**

### 기술적 원리

1. **실행 장소:** 함수에서 실행한 `print` 함수는 Driver가 아니라 **각 Worker 노드(컨테이너)** 에서 실행됩니다.
2. **출력 장소:** 출력 결과(`stdout`)는 Worker 컴퓨터의 **내부 로그 파일**에 기록됩니다. Driver에게 전송되지 않습니다.

> **해결책:** 눈으로 꼭 봐야겠다면 `rdd.take(5).foreach(print)` 처럼 소량만 가져와서(`take`) 내 앞에서 출력하거나, 아래의 **로그 확인법**을 써야 합니다.

---
## `foreach` vs `foreachPartition` 

실무(DB 저장, API 호출)에서는 `foreach`보다 **`foreachPartition`** 을 훨씬 많이 씁니다.

### ① `foreach(f)` : 하나씩 처리 (Row-by-Row)

- **특징:** 데이터 1건마다 함수 호출.
- **단점:** DB 연결을 한다면? 100만 건 데이터에 대해 **DB 연결을 100만 번** 맺고 끊습니다. (서버 폭발 )

```python
# ❌ [비효율]
rdd.foreach(lambda row: db.save(row))
```

### ② `foreachPartition(f)` : 뭉텅이로 처리 (Batch) 

- **특징:** 파티션(Partition) 1개를 통째로 던져줍니다.
- **장점:** **DB 연결을 파티션당 1번만** 하면 됩니다. (100만 건이라도 파티션이 5개면 연결 5번!)

```python
def process_partition(iterator):
    # 1. (파티션 당 1회) DB 연결 맺기 
    # connection = create_db_connection() 
    
    # 2. 파티션 내의 데이터들을 반복문으로 처리
    for row in iterator:
        # connection.save(row)
        pass
        
    # 3. (파티션 당 1회) DB 연결 끊기
    # connection.close()

# ✅ [권장] 실무용 코드
rdd.foreachPartition(process_partition)
```

---
## 숨겨진 로그 찾아보기 (Docker) 

"옆방 TV가 켜졌는지 확인하는 법"입니다. Worker 컨테이너의 로그를 뒤져야 합니다.

### Step 1. 워커 로그 겉핥기

```bash
docker logs spark-worker
# "INFO ExecutorRunner: Launch command..." 가 보이면 일꾼이 뭔가 하고 있다는 뜻!
```

### Step 2. 진짜 출력물(stdout) 찾기

```bash
# 1. 워커 컨테이너 접속
docker exec -it spark-worker /bin/bash

# 2. 작업 폴더로 이동 (app-아이디/일꾼번호)
cd /opt/spark/work/app-2026xxxx-xxxx/0

# 3. 출력 파일 확인
cat stdout
```

---
## 상황별 추천 (Best Practice)

|**목적**|**추천 방식**|**이유**|
|---|---|---|
|**눈으로 확인하고 싶을 때**|`rdd.take(5)`|5개만 가져오니 안전함. (맛보기)|
|**최종 결과가 작을 때**|`collect()`|리스트로 변환해서 파이썬으로 처리하기 편함.|
|**최종 결과가 클 때**|`saveAsTextFile()`|화면에 찍으면 멈춤. 파일 저장이 정석.|
|**DB/API에 저장할 때**|**`foreachPartition()`**|연결 비용(Connection Cost) 절약의 핵심.|