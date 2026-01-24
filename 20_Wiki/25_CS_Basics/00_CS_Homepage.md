

##  핵심 엔지니어링 원칙 (Engineering Principles)

> 개발자가 갖춰야 할 사고방식 
> 
- [[Idempotent(멱등성)]] :  100번 실행해도 결과는 같아야 한다. (데이터 엔지니어링 제1원칙) 
- [[Trade_off]] : 공짜 점심은 없다. (속도 vs 안정성, 무엇을 포기할 것인가?) 
- [[Abstraction]] : 복잡한 건 숨기고 단순하게 보여주기 (함수, 클래스, API의 본질)

##  운영체제 & 시스템 (OS & System) 

> 내 코드가 돌아가는 환경 이해하기

- [[Process_vs_Thread]] : 프로그램(Process)과 일꾼(Thread)의 차이 
- [[Concurrency_vs_Parallelism]] : 동시성(빠르게 번갈아 하기) vs 병렬성(진짜 동시에 하기) 
- [[Blocking_vs_NonBlocking]] : "기다릴까(Blocking), 아니면 딴 거 하고 있을까(Non-Blocking)?"

##  네트워크 & 통신 (Network)

> 데이터가 이동하는 길

- [[HTTP_Basics]] : 웹 통신의 기본 약속 (Request/Response)
- [[REST_API_Methods]] : GET, POST, PUT, DELETE의 정확한 의미와 멱등성 차이
- [[TCP_IP]] : 인터넷이 데이터를 안 흘리고 배달하는 방법

##  데이터베이스 이론 (Database)

> SQL 문법을 넘어선 데이터 관리 이론

- [[Transaction_ACID]] : "송금하다가 에러 나면 돈은 증발하나?" (트랜잭션의 4가지 성질)
- [[Index_Concept]] : 책의 '색인'처럼 데이터 빨리 찾는 법
- [[CAP_Theorem]] : 분산 시스템의 딜레마 (일관성 vs 가용성)

##  자료구조 & 알고리즘 (Data Structure)

> 효율적인 코드 작성의 기초

- [[Big_O_Notation]] : "이 코드, 데이터 100만 개 넣으면 멈출까요?" (시간 복잡도)
- [[CS_Stack_Queue]] : 접시 쌓기(LIFO) vs 맛집 줄 서기(FIFO)
- [[Hash_Map]] : 검색 속도를 광속으로 만드는 마법 (Key-Value)