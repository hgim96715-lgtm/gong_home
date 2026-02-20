

##  핵심 엔지니어링 원칙 (Engineering Principles)

> 개발자가 갖춰야 할 사고방식 
> 
- [[Idempotent(멱등성)]] :  100번 실행해도 결과는 같아야 한다. (데이터 엔지니어링 제1원칙) 
- [[Trade_off]] : 공짜 점심은 없다. (속도 vs 안정성, 무엇을 포기할 것인가?) 
- [[Abstraction]] : 복잡한 건 숨기고 단순하게 보여주기 (함수, 클래스, API의 본질)

##  운영체제 & 시스템 (OS & System) 

> 내 코드가 돌아가는 환경 이해하기

- [[CPU_Architecture]] : 코어(Core)와 스레드, 그리고 부하(Load)의 관계
- [[RAM_Memory]] : 하드디스크는 '도서관', RAM은 '책상'. (속도와 용량의 차이)
- [[Process_vs_Thread]] : 프로그램(Process)과 일꾼(Thread)의 차이 
- [[Concurrency_vs_Parallelism]] : 동시성(빠르게 번갈아 하기) vs 병렬성(진짜 동시에 하기) 
- [[Blocking_vs_NonBlocking]] : "기다릴까(Blocking), 아니면 딴 거 하고 있을까(Non-Blocking)?"



## 분산 시스템 및 인프라 (Distributed Systems & Infra)

> "컴퓨터 한 대로는 부족할 때, 여러 대를 팀으로 만드는 기술"

- **[[Concept_Cluster]]** : 컴퓨터 여러 대를 한 팀으로 만드는 원리 (Master & Worker, 이사 팀 비유)
- **[[Concept_Node_Scaling]]** : "더 큰 컴퓨터를 살까(Scale-up), 컴퓨터를 더 살까(Scale-out)?"
- **[[Load_Balancing]]** : 들어오는 요청을 여러 서버에 골고루 나눠주기 (교통 정리)
- **[[Throttling_RateLimiting]] : "서버가 안 터지게 신호등 설치하기 (속도 조절과 백프레셔)"**





##  네트워크 & 통신 (Network)

> 데이터가 이동하는 길

- [[HTTP_Basics]] : 웹 통신의 기본 약속 (Request/Response)
- [[REST_API_Methods]] : GET, POST, PUT, DELETE의 정확한 의미와 멱등성 차이
- [[TCP_IP]] : 인터넷이 데이터를 안 흘리고 배달하는 방법


## 데이터 표현과 포맷 (Data Representation)

> "0과 1을 어떻게 문자와 파일로 만들 것인가?"

- [[Text_vs_Binary]] : 파일의 두 종류, 텍스트(Text)와 바이너리(Binary)의 차이 (r vs rb)
- [[Data_Formats_CSV]] : 데이터 분석의 쌀, CSV의 구조와 주의점 (구분자, 이스케이프), "대표적인 텍스트 파일"
- [[Encoding_Concept]] : "한글이 왜 깨지나요?" (ASCII, UTF-8, CP949의 원리) `Encoding`,`Decoding`
- [[Data_Formats_Parquet]] : CSV보다 10배 빠른 빅데이터 전용 포맷, "대표적인 바이너리 파일"
- [[Serialization_JSON_XML]] :  사람이 읽기 편한 데이터 (Key-Value 텍스트)


##  데이터베이스 이론 (Database)

> SQL 문법을 넘어선 데이터 관리 이론

- [[Transaction_ACID]] : "송금하다가 에러 나면 돈은 증발하나?" (트랜잭션의 4가지 성질)
- [[Index_Concept]] : 책의 '색인'처럼 데이터 빨리 찾는 법
- [[Node_vs_Partition]] : "피자 가게(Node)와 피자 조각(Partition)의 관계." (물리적 서버 vs 논리적 데이터)
- [[CAP_Theorem]] : 분산 시스템의 딜레마 (일관성 vs 가용성)

##  자료구조 & 알고리즘 (Data Structure)

> 효율적인 코드 작성의 기초

- [[Big_O_Notation]] : "이 코드, 데이터 100만 개 넣으면 멈출까요?" (시간 복잡도)
- [[CS_Stack_Queue]] : 접시 쌓기(LIFO) vs 맛집 줄 서기(FIFO)
- [[Hash_Map]] : 검색 속도를 광속으로 만드는 마법 (Key-Value

