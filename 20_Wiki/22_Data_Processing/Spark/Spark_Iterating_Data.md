---
aliases:
  - foreach
  - collect
  - 스파크반복문
  - OOM
  - Driver_vs_Executor
tags:
  - Spark
  - Performance
  - Coding
  - BestPractice
related:
  - "[[Spark_Architecture]]"
  - "[[Transformations_vs_Actions]]"
  - "[[Python_Control_Flow]]"
  - "[[00_Apache_Spark_HomePage]]"
---
## 개념 한 줄 요약

**"데이터를 내 앞으로 가져올래(`collect`), 아니면 내가 데이터한테 갈까(`foreach`)?"**

* **`collect()` + `for`:** 모든 데이터를 **Driver(반장)** 에게 가져와서 반장이 혼자 하나씩 처리함. (병목 현상 )
* **`foreach()`:** 처리해야 할 로직(함수)을 **Executor(일꾼)** 들에게 보내서 각자 알아서 처리하게 시킴. (병렬 처리 ️)

---
## 죽음의 반복문: `collect()`

사용자님이 작성하신 코드는 **"작은 데이터(결과 요약)"** 를 볼 때만 써야 합니다.

```python
# [위험한 코드]
# 1. 1TB 데이터를 Driver 메모리로 전부 로딩 (여기서 이미 컴퓨터 멈춤)
# 2. 파이썬 싱글 스레드로 하나씩 출력 (엄청 느림)
for datum in rdd.collect():
    print(datum)
```

**언제 쓰는가?**

- `word_count` 결과처럼 데이터가 확 줄어들어서 한 눈에 볼 수 있을 때. (수천 건 이내)
- 디버깅용으로 `take(5)` 한 데이터를 볼 때.

---
## 3. 진정한 분산 반복문: `foreach()` 

데이터가 아무리 커도 터지지 않는 방법입니다. **Action(활동조)** 함수입니다.

```python
# [올바른 코드]
# Driver가 아니라 각 Worker 노드에서 병렬로 실행됨
def print_data(x):
    # flush=True : 버퍼링 없이 즉시 출력해라! (중요)
    print(x, flush=True)

# counts는 아까 만든 RDD 변수 이름!
counts.foreach(print_data)

# 또는 람다(Lambda)로 한 줄 컷
counts.foreach(lambda x: print(x, flush=True))
```

### 초보자의 흔한 실수: "NameError가 떠요!"

예제를 복사해서 쓰다 보면 이런 실수를 많이 합니다.

- **상황:** 내 변수 이름은 `counts`인데, 예제 코드의 `rdd`를 그대로 씀.
- **에러:** `NameError: name 'rdd' is not defined`
- **해결:** **"내가 방금 만든 변수 이름이 뭐였지?"** 확인하고 이름을 맞춰줘야 합니다.

### "왜 `foreach`로 `print`하면 화면에 안 나와요?"

Jupyter에서 `rdd.foreach(print)`를 실행하면 에러도 안 나는데 **화면에 아무것도 출력되지 않습니다.**

- **이유:**
    - `print` 함수가 실행되는 장소는 **Jupyter(Driver)** 가 아니라 저 멀리 있는 **Worker(Container)** 이기 때문입니다.
    - 출력 결과는 Jupyter 화면이 아니라, **Worker 컨테이너의 로그 파일(stdout,stderr)** 에 찍히고 있습니다.

### 그럼 다른건 왜 ?

**`counts.take(2)`**:

- Worker에 있는 데이터 2개를 **Driver(Jupyter)로 가져옵니다.**
- Jupyter가 데이터를 받았으니 화면에 출력해줍니다. (`[(word, 1), (test, 1)]`)

---
## [심화] 터미널로 로그 탐험하기 (Under the Hood)

Jupyter 화면에는 안 나오지만, Worker들이 몰래 써둔 일기장을 훔쳐보는 방법입니다. (구조를 이해하는 데 아주 좋습니다!)

### Step 1. 매니저(Worker)의 로그 보기 (간단 버전)

일단 Worker가 일을 하고 있는지 겉에서 확인합니다.

```bash
docker logs spark-worker
```

**확인 포인트:**

- `INFO Worker: Asked to launch executor...` : 마스터가 일꾼 좀 뽑으라고 시켰음.
- **`INFO ExecutorRunner: Launch command...` : 실제로 일꾼(Java 프로세스)을 실행시킴! (이게 보이면 성공)**

### Step 2. 일꾼(Executor)의 방으로 쳐들어가기 (심화 버전) 

진짜 `print` 결과는 이 깊은 곳에 숨겨져 있습니다.

**1. 워커 컨테이너 접속:**

```bash
docker exec -it spark-worker /bin/bash
```

**2.작업실 이동: (여기에 앱 ID로 된 폴더들이 있음)**

```bash
cd /opt/spark/work
ls
```

(여기서 `app-2026...` 처럼 생긴 폴더가 보일 겁니다. 방금 로그에서 본 ID입니다!)

**3.내 앱 찾기: (방금 실행한 시간대의 폴더로 이동)**

```bash
cd app-2026xxxx-xxxx  # (탭 키 누르면 자동완성 됨)
cd 0                  # 0번 일꾼의 방
```

**연습장 훔쳐보기:**

```bash
cat stdout   # (정상 출력 내용)
cat stderr   # (에러나 잡다한 로그들 - 여기에 print가 섞이기도 함)
```

**💡 깨달음:** "아, 내가 `foreach(print)`를 하면 내 컴퓨터에 찍히는 게 아니라, 
저 멀리 있는 **컨테이너 구석탱이 파일(`stdout`)** 에 글자가 적히는구나!"


---
## 5. 상황별 추천 (Best Practice) 🏆

| **목적** | **추천 방식** | **이유** |
| :--- | :--- | :--- |
| **눈으로 확인하고 싶을 때** | `rdd.take(5)` | 5개만 가져오니 안전함. (맛보기) |
| **최종 결과가 작을 때** | `collect()` | 리스트로 변환해서 파이썬으로 지지고 볶기 편함. |
| **최종 결과가 너무 클 때** | **`saveAsTextFile()`** | **화면에 찍으면 브라우저 멈춤(OOM). 파일로 저장하는 게 정석.** |
| **DB에 저장할 때** | `foreach()` | 각 Worker가 동시에 DB에 접속해서 데이터를 밀어넣어야 빠름. |
| **API 요청 보낼 때** | `foreach()` | 병렬로 요청을 쏴야 빠름. |

### 💡 `saveAsTextFile` 사용 시 주의사항

```python
# data 폴더에 결과 파일 저장 (Driver가 아니라 Worker가 씀)
counts.saveAsTextFile("file:///workspace/data/wordcount_result")
```

- **주의:** 저장하려는 폴더(`wordcount_result`)가 **이미 존재하면 에러**(`FileAlreadyExistsException`)가 납니다.
- **해결:** 매번 다른 이름을 쓰거나, 실행 전에 해당 폴더를 지워야 합니다.


>**작으면 `collect`, 크면 `save`, 외부 전송은 `foreach`.**

---
### Tip

"초보 때는 `foreach`를 쓸 일이 별로 없어.
왜냐하면 우린 주로 결과를 눈으로 보고 싶어 하니까(`collect`). 
하지만 나중에 **'이 데이터를 Redis나 MySQL에 저장해라'** 같은 미션이 떨어지면, 그때 `collect()`를 쓰면 네 컴퓨터가 폭발할 거야.
그때 꼭 **`foreach`** (또는 `foreachPartition`)를 기억해내야 해!"

