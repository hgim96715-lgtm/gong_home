---
aliases:
  - Outer Join Limitations
  - Streaming Left Join
  - 아우터 조인 한계
  - Stream-Stream Outer Join
tags:
  - Streaming
  - Spark
related:
  - "[[Spark_Streaming_Stream_Join]]"
  - "[[Spark_Streaming_Watermark]]"
---
# Streaming Outer Join의 한계와 필수 조건

##  Outer Join이란?

* **Inner Join:** 두 데이터가 모두 도착해야 결과가 나옴 (교집합).
* **Outer Join (Left/Right):** 짝이 안 와도 기준이 되는 데이터는 무조건 살림.
    * 예: "광고는 노출됐는데(`Left`), 클릭(`Right`)이 안 온 경우"를 찾아내어 `click_uuid`를 `NULL`로 채워서 출력.

---
##  치명적인 한계점 (Limitations)

Streaming Outer Join은 Inner Join과 달리 **"기다림의 끝"** 을 알 수 없다는 구조적 문제가 있습니다.

### ① 무한 대기 (The Infinity Problem)

Inner Join은 짝이 오면 내보내고 지우면 되지만, Outer Join은 **"클릭이 안 온 건지, 오고 있는데 늦는 건지"** 알 방법이 없습니다. 스파크는 기본적으로 영원히 기다립니다.

### ② 메모리 폭발 위험 (OOM)

영원히 기다린다는 뜻은 **State Store(메모리)** 에 데이터를 영원히 쌓아둔다는 뜻입니다. 
결국 메모리 초과(OOM) 에러가 발생합니다.

### ③ 결과 출력의 지연 (Output Delay) 🐢

이게 가장 중요합니다. **매칭되지 않은 데이터(`Null` 결과)** 는 바로 나오지 않습니다.

* **매칭 성공:** 즉시 출력.
* **매칭 실패(Null):** **워터마크 시간 + 시간 제약 조건**이 지날 때까지 기다렸다가, "아 이제 진짜 안 오네"라고 확신할 때 비로소 출력됩니다.

---
##  필수 해결책: 2가지 강제 규칙

이 문제를 막기 위해 스파크는 Outer Join 시 **두 가지 조건을 강제**합니다.
(하나라도 빠지면 `AnalysisException` 에러 발생)

### 1) 워터마크(Watermark) 필수

양쪽 스트림(Left, Right) 모두에 `.withWatermark()`가 있어야 합니다.

### 2) 시간 제약(Event-Time Constraints) 필수

조인 조건(`ON` 또는 `WHERE`)에 반드시 **시간 범위**가 포함되어야 합니다.
> **의미:** "이 시간 범위(예: 5분)가 지나면 더 이상 기다리지 말고 `NULL`로 출력하고 메모리에서 지워라!"

---
## [실습] 코드 분석 및 적용

규칙을 준수한 모범 코드입니다.

```python
# ... (SparkSession 생성 생략) ...

# 1. Impression 데이터 (Left Side)
impressions_df = impression_events.select(...) \
    .withColumnRenamed("create_date", "impr_date") \
    .withWatermark("impr_date", "5 minutes") \ # ✅ 필수 1: 워터마크
    .withColumnRenamed("uuid", "impr_uuid")

# 2. Click 데이터 (Right Side)
clicks_df = click_events.select(...) \
    .withColumnRenamed("create_date", "click_date") \
    .withWatermark("click_date", "5 minutes") \ # ✅ 필수 1: 워터마크
    .withColumnRenamed("uuid", "click_uuid")

# 3. Left Outer Join 수행
join_df = impressions_df.join(
    clicks_df,
    expr("""
        impr_uuid = click_uuid AND
        click_date >= impr_date AND
        click_date <= impr_date + interval 5 minutes
        """), # ✅ 필수 2: 시간 제약 (Event-Time Constraint)
    "leftOuter" # 👈 Left Join: 클릭 안 한 사람도 결과에 나옴 (지연 출력됨)
).drop("click_uuid").drop(clicks_df.placement_id)
```

---
## 결과 확인 시나리오 (타이밍 주의)

Outer Join 테스트 시 가장 많이 당황하는 부분입니다.

| **상황**   | **입력 데이터**         | **동작 결과**                                                                                |
| -------- | ------------------ | ---------------------------------------------------------------------------------------- |
| **상황 A** | 광고 노출 후 1분 뒤 클릭 도착 | **즉시 출력** (Inner Join과 동일)                                                               |
| **상황 B** | 광고 노출됨. 클릭은 안 함    | **즉시 안 나옴!** 🙅‍♂️<br><br>워터마크(5분) + 간격(5분) = **약 10분 뒤**에 `click` 컬럼이 `NULL`로 채워져서 출력됨. |

**💡 결론:** "클릭 안 한 사람 데이터가 왜 안 나오지?"라고 생각하지 마세요. 
스파크가 **"혹시 늦게 클릭할까 봐"** 기다리는 중입니다. 워터마크 시간이 지나면 나옵니다!