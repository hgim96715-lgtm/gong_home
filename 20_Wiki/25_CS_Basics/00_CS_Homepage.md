
> 배워 나가야하는 CS 지식

---
---

## 1. 운영체제 & 시스템 (OS & System)

```
내 코드가 돌아가는 환경 이해하기
리눅스 마스터 1급 + 데이터 엔지니어 실무 기초
```

|노트|핵심 개념|
|---|---|
|[[CS_Operating_System]]|OS 역할 / 프로세스·스레드 / 스케줄링 / 메모리 / inode / 데드락|
|[[CS_CPU_Architecture]]|코어 / 스레드 / Load Average / 클럭 속도|
|[[CS_RAM_Memory]]|휘발성 / 속도 차이 / OOM / 스왑 메모리|
|[[Process_vs_Thread]]|PID / 컨텍스트 스위칭 / 공유 메모리 / GIL|
|[[Concurrency_vs_Parallelism]]|동시성 vs 병렬성 / async / 멀티코어|
|[[CS_Blocking_vs_NonBlocking]]|Blocking / Non-Blocking / Async·Await / I/O 대기|

> 리눅스 실습 → [[00_Linux_HomePage]]

---

---

## 2. 분산 시스템 & 인프라

```
컴퓨터 한 대로 부족할 때, 여러 대를 팀으로 만드는 기술
Kafka / Spark / Docker 이해의 기초
```

|노트|핵심 개념|
|---|---|
|[[Concept_Cluster]]|Master & Worker / 클러스터 / 장애 허용|
|[[Concept_Node_Scaling]]|Scale-up vs Scale-out / 수직·수평 확장 / 비용 비교|
|[[Load_Balancing]]|Round Robin / Least Connection / Health Check|
|[[Throttling_RateLimiting]]|Throttling / Rate Limiting / 백프레셔 / 429|
|[[Buffering_vs_Batching]]|Buffer / Batch / I/O 최적화 / 메모리 vs 처리량|

---

---

## 3. 네트워크 & 통신

```
데이터가 이동하는 길
API 연동 / 포트 / 프로토콜 이해
```

|노트|핵심 개념|
|---|---|
|[[CS_HTTP_Basics]]|Request / Response / Header / Body / Status Code / 무상태성|
|[[CS_REST_API_Methods]]|GET / POST / PUT / DELETE / PATCH / 멱등성 / endpoint|
|[[CS_TCP_IP]]|TCP / UDP / 3-Way Handshake / 패킷 / 포트 / 소켓|

---

---

## 4. 데이터 표현과 포맷

```
0과 1을 어떻게 문자와 파일로 만드는가
직렬화 / 인코딩 / 파일 포맷 이해
```

|노트|핵심 개념|
|---|---|
|[[Text_vs_Binary]]|텍스트 파일 vs 바이너리 파일 / r vs rb / hex 덤프|
|[[Encoding_Decoding_Concept]]|ASCII / UTF-8 / CP949 / Encoding / Decoding / BOM|
|[[Serialization_JSON_XML]]|직렬화·역직렬화 / JSON / XML / Key-Value|
|[[Data_Formats_CSV]]|구분자 / 이스케이프 / 헤더 / 개행 문자|
|[[Data_Formats_Parquet]]|컬럼 기반 저장 / 압축 / 스키마 / CSV 대비 성능|

---

---

## 5. 데이터베이스 이론

```
SQL 문법을 넘어선 데이터 관리 이론
트랜잭션 / 인덱스 / 분산 DB 설계
```

|노트|핵심 개념|
|---|---|
|[[Transaction_ACID]]|Atomicity / Consistency / Isolation / Durability / COMMIT / ROLLBACK|
|[[Index_Concept]]|B-Tree / Full Scan vs Index Scan / 복합 인덱스 / 카디널리티|
|[[Node_vs_Partition]]|Node / Partition / 샤딩 / 물리 서버 vs 논리 데이터|
|[[CAP_Theorem]]|Consistency / Availability / Partition Tolerance / CP vs AP|

---

---

## 6. 자료구조 & 알고리즘

```
효율적인 코드 작성의 기초
시간 복잡도 이해 / 자주 쓰이는 자료구조
```

|노트|핵심 개념|
|---|---|
|[[Big_O_Notation]]|O(1) / O(n) / O(log n) / O(n²) / 시간·공간 복잡도|
|[[CS_Stack_Queue]]|Stack (LIFO) / Queue (FIFO) / push / pop / Deque|
|[[Hash_Map]]|Hash / Key-Value / 충돌 해결 / O(1) 검색 / 딕셔너리|

---

---

## 7. 수학 & 선형대수 기초

```
sklearn / numpy / spark MLlib 이해의 핵심 전제지식
TF-IDF, cosine_similarity 가 왜 그렇게 동작하는지 이해하기
```

|노트|핵심 개념|
|---|---|
|[[CS_Vector_Matrix]]|벡터란 / 행렬이란 / 방향과 크기 / 1차원 vs 2차원 / TF-IDF 행렬 구조 / matrix[0] vs matrix[0:1]|
|[[Numpy_Sparse_Matrix]]|희소 행렬 / (행, 열) 좌표 저장 / CSR 형식 / stored elements 읽기 / .toarray()|

> 실전 적용 → [[Sklearn_TF_IDF]], [[Sklearn_Cosine_Similarity]]

---

---

## 8. 핵심 엔지니어링 원칙

```
개발자가 갖춰야 할 사고방식
설계 결정의 기준이 되는 원칙들
```

|노트|핵심 개념|
|---|---|
|[[Idempotent(멱등성)]]|100번 실행해도 결과는 같아야 한다 / PUT vs POST / 데이터 엔지니어링 제1원칙|
|[[Trade_off]]|속도 vs 안정성 / 비용 vs 성능 / 공짜 점심은 없다|
|[[Abstraction]]|복잡한 건 숨기고 단순하게 / 함수 / 클래스 / API / 캡슐화|

---