
# [Airflow + Kafka + Flink + Spark + Cassandra + MinIO] 환경 구축

  
이 프로젝트는 배치 + 스트리밍 데이터 파이프라인 실습 및 통합 테스트를 위한 로컬 개발 환경입니다.
Docker 기반으로 구성 되었습니다.
Docker compose 구상은 각각의 docker compose에서 알아 차린 것으로 구상해서 다시 만들었습니다.

  
  ---
  
## 구성 요소

| 컴포넌트          | 역할              |
| ------------- | --------------- |
| **Airflow**   | 워크플로우 관리 및 스케줄링 |
| **Kafka**     | 실시간 메시지 브로커     |
| **Flink**     | 실시간 스트리밍 처리     |
| **Spark**     | 대용량 배치 처리       |
| **Cassandra** | 분산 NoSQL 저장소    |
| **MinIO**     | S3 호환 오브젝트 스토리지 |

  
---


## Dashboard


### Airflow (워크플로우 관리)

* 주소: [airflow_login](http://localhost:8080/login/)

* 로그인: `airflow / airflow`  

### Flink (실시간 처리)

* 주소: [flink](http://localhost:8081)

### Spark Master (대용량 배치 처리)

* 주소: [spark_master](http://localhost:9090)

### MinIO (S3 스토리지)

* 주소: [minIO](http://localhost:9001)

* 로그인: `ROOTNAME / CHANGEME123`


---

## requiremnets 분리


각 컴포넌트는 **실행 환경과 역할이 서로 다르기 때문에** Python 의존성을 분리하여 관리합니다.

1. 버전 충돌을 방지, 
2. 컨테이너별로 가볍고 안정적 실행환경 유지

- `requirements.txt` : Airflow (워크플로우 오케스트레이션)
- `requirements_spark.txt` : Spark (배치 처리 / PySpark Job)
- `requirements_flink.txt` : Flink (스트리밍 처리 / PyFlink Job)