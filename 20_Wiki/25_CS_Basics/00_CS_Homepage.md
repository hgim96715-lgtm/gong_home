## 핵심 엔지니어링 원칙 (Engineering Principles)

> 개발자가 갖춰야 할 사고방식

- [[Idempotent(멱등성)]] : 100번 실행해도 결과는 같아야 한다. (`멱등성`, `PUT vs POST`, `중복 실행 안전성`, `데이터 엔지니어링 제1원칙`)
- [[Trade_off]] : 공짜 점심은 없다. (`속도 vs 안정성`, `비용 vs 성능`, `무엇을 포기할 것인가?`)
- [[Abstraction]] : 복잡한 건 숨기고 단순하게 보여주기 (`함수`, `클래스`, `API`, `캡슐화`, `인터페이스`)

---

## 운영체제 & 시스템 (OS & System)

> 내 코드가 돌아가는 환경 이해하기

- [[CPU_Architecture]] : 코어(Core)와 스레드, 그리고 부하(Load)의 관계 (`Core`, `Thread`, `Load Average`, `클럭 속도`)
- [[RAM_Memory]] : 하드디스크는 '도서관', RAM은 '책상' (`휘발성`, `속도 차이`, `OOM`, `스왑 메모리`)
- [[Process_vs_Thread]] : 프로그램(Process)과 일꾼(Thread)의 차이 (`PID`, `컨텍스트 스위칭`, `공유 메모리`, `GIL`)
- [[Concurrency_vs_Parallelism]] : 동시성 vs 병렬성 (`Concurrency`, `Parallelism`, `async`, `멀티코어`, `빠르게 번갈아 하기`)
- [[Blocking_vs_NonBlocking]] : 기다릴까, 딴 거 하고 있을까? (`Blocking`, `Non-Blocking`, `Async/Await`, `I/O 대기`)

---

## 분산 시스템 및 인프라 (Distributed Systems & Infra)

> "컴퓨터 한 대로는 부족할 때, 여러 대를 팀으로 만드는 기술"

- **[[Concept_Cluster]]** : 컴퓨터 여러 대를 한 팀으로 만드는 원리 (`Master & Worker`, `클러스터`, `장애 허용`, `이사 팀 비유`)
- **[[Concept_Node_Scaling]]** : 더 큰 컴퓨터를 살까, 컴퓨터를 더 살까? (`Scale-up`, `Scale-out`, `수직 확장`, `수평 확장`, `비용 비교`)
- **[[Load_Balancing]]** : 들어오는 요청을 여러 서버에 골고루 나눠주기 (`Round Robin`, `Least Connection`, `Health Check`, `교통 정리`)
- **[[Throttling_RateLimiting]]** : 서버가 안 터지게 신호등 설치하기 (`Throttling`, `Rate Limiting`, `백프레셔`, `429 Too Many Requests`)
- [[Buffering_vs_Batching]] : 한 개씩 옮길래, 꽉 채워서 한 번에 옮길래? (`Buffer`, `Batch`, `I/O 최적화`, `메모리 vs 처리량`)

---

## 네트워크 & 통신 (Network)

> 데이터가 이동하는 길

- [[HTTP_Basics]] : 웹 통신의 기본 약속 (`Request`, `Response`, `Header`, `Body`, `Status Code`, `무상태성`)
- [[REST_API_Methods]] : GET, POST, PUT, DELETE의 정확한 의미와 멱등성 차이 (`GET`, `POST`, `PUT`, `DELETE`, `PATCH`, `멱등성`)
- [[TCP_IP]] : 인터넷이 데이터를 안 흘리고 배달하는 방법 (`TCP`, `UDP`, `3-Way Handshake`, `패킷`, `포트`)

---

## 데이터 표현과 포맷 (Data Representation)

> "0과 1을 어떻게 문자와 파일로 만들 것인가?"

- [[Text_vs_Binary]] : 파일의 두 종류, 텍스트와 바이너리의 차이 (`r vs rb`, `텍스트 파일`, `바이너리 파일`, `hex 덤프`)
- [[Data_Formats_CSV]] : 데이터 분석의 쌀, CSV의 구조와 주의점 (`구분자`, `이스케이프`, `헤더`, `개행 문자`, `대표적인 텍스트 파일`)
- [[Encoding_Concept]] : "한글이 왜 깨지나요?" (`ASCII`, `UTF-8`, `CP949`, `Encoding`, `Decoding`, `BOM`)
- [[Data_Formats_Parquet]] : CSV보다 10배 빠른 빅데이터 전용 포맷 (`컬럼 기반 저장`, `압축`, `스키마`, `대표적인 바이너리 파일`)
- [[Serialization_JSON_XML]] : 사람이 읽기 편한 데이터 (`Key-Value`, `직렬화`, `역직렬화`, `JSON`, `XML`, `스키마 없음`)

---

## 데이터베이스 이론 (Database)

> SQL 문법을 넘어선 데이터 관리 이론

- [[Transaction_ACID]] : "송금하다가 에러 나면 돈은 증발하나?" (`Atomicity`, `Consistency`, `Isolation`, `Durability`, `COMMIT`, `ROLLBACK`)
- [[Index_Concept]] : 책의 '색인'처럼 데이터 빨리 찾는 법 (`B-Tree`, `Full Scan vs Index Scan`, `복합 인덱스`, `카디널리티`)
- [[Node_vs_Partition]] : 피자 가게(Node)와 피자 조각(Partition)의 관계 (`Node`, `Partition`, `샤딩`, `물리적 서버 vs 논리적 데이터`)
- [[CAP_Theorem]] : 분산 시스템의 딜레마 (`Consistency`, `Availability`, `Partition Tolerance`, `CP vs AP`)

---

## 자료구조 & 알고리즘 (Data Structure)

> 효율적인 코드 작성의 기초

- [[Big_O_Notation]] : "이 코드, 데이터 100만 개 넣으면 멈출까요?" (`O(1)`, `O(n)`, `O(log n)`, `O(n²)`, `시간 복잡도`, `공간 복잡도`)
- [[CS_Stack_Queue]] : 접시 쌓기(LIFO) vs 맛집 줄 서기(FIFO) (`Stack`, `Queue`, `push`, `pop`, `dequeue`, `Deque`)
- [[Hash_Map]] : 검색 속도를 광속으로 만드는 마법 (`Hash`, `Key-Value`, `충돌 해결`, `O(1) 검색`, `딕셔너리`)