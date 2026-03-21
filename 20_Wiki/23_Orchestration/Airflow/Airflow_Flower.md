---
aliases:
  - Flower
  - Celery Monitoring
  - 워커 모니터링
  - Airflow Flower
tags:
  - Airflow
related:
  - "[[Docker_Host_Access]]"
  - "[[Airflow_Architecture]]"
  - "[[Airflow_Queues_Scaling]]"
---
## 개념 한 줄 요약

**Flower**는 Airflow의 워커(Worker)들이 일을 잘하고 있는지 감시하는 **"웹 기반 CCTV 모니터링 툴"** 이야. 
정확히는 **Celery Cluster** 관리 도구지.

---
## 왜 필요한가 (Why)

Airflow UI만으로는 부족한 순간이 꼭 와.

 **문제점:** 
 - Airflow UI는 태스크가 "성공/실패"했는지는 보여주지만, **"워커가 살았는지 죽었는지"** 혹은 **"워커가 메모리를 얼마나 잡아먹고 있는지"** 는 안 보여줘.
 - 태스크가 `Queued` 상태에서 멈춰있으면 답답해 미치지.

**해결책:** 
- Flower를 켜면 현재 실행 중인 워커의 리소스 상태, 큐(Queue)에 쌓인 작업량, 그리고 워커를 강제로 껐다 켜는 관리까지 가능해.

---
## Practical Context (실무 활용)

현업에서 이럴 때 Flower를 켭니다.

1.  **Tasks stuck in Queued:** 
	- 태스크가 실행되지 않고 `Queued` 상태로 영원히 멈춰있을 때 (워커가 죽었거나 꽉 찼는지 확인).
2.  **Worker Resource Monitoring:** 특정 DAG만 돌면 워커가 자꾸 죽을 때 (메모리 누수 확인).
3.  **Worker Management:** 꼬인 워커를 웹에서 바로 재시작(Restart)하고 싶을 때.

---
##  Code Core Points

Flower는 보통 항상 켜두기보다는 필요할 때 켜는 경우가 많아 (리소스 절약). 
도커 환경에서는 별도의 프로필(`--profile flower`) 옵션을 줘야 실행돼.

### Docker Compose로 실행하기

```bash
# 기본적으로 Airflow 컨테이너들은 띄워져 있는 상태에서,
# Flower만 추가로 띄우고 싶을 때 사용
# 처음부터 Flower까지 쓰고 싶을때도 이거 사용해도 상관 없음 
# 하지만 평소에는 docker compose up -d 이렇게 하고 ,필요할때 실행 
docker compose --profile flower up -d
```

- **`--profile flower`**: `docker-compose.yaml` 파일에 `flower` 서비스가 정의되어 있어도, 프로필을 지정하지 않으면 실행되지 않게 막혀 있는 경우가 많아. 이걸 풀어주는 옵션이야.
- **접속 주소:** 보통 `localhost:5555` 포트를 사용해.

---
## 상세 분석  (Flower 기능)

Flower 화면에 들어가면 볼 수 있는 핵심 정보들이야.

1. **Worker Status:** 워커가 온라인인지 오프라인인지 한눈에 보임.
2. **Inspect Active Tasks:** 현재 워커가 처리 중인 태스크가 무엇이고, 어떤 인자(Arguments)를 받았는지, 실행된 지 얼마나 됐는지(Runtime) 확인 가능.
3. **Monitor Queues:** Celery 큐에 대기 중인 작업이 얼마나 쌓여있는지 확인.
4. **Manage Workers:** 터미널 접속 없이 웹 UI에서 워커 프로세스를 정지(Stop)하거나 재시작(Restart) 가능.

---
## 초보자가 자주 착각하는 포인트

1.  **"LocalExecutor에서도 쓰나요?"**
    - 아니! Flower는 **`CeleryExecutor`** (분산 처리 환경)를 쓸 때만 필요해. 
    - 혼자서 다 하는 `LocalExecutor`나 `SequentialExecutor`에서는 Flower가 뜰 이유도 없고 뜰 수도 없어.

2.  **"Airflow UI랑 같은 거 아닌가요?"**
    - 완전 달라. 
    - Airflow UI는 "업무 일지(DAG Run)"를 보는 곳이고, 
    - Flower는 "직원들의 건강 상태(Worker Health)"를 보는 곳이야.

3.  **"Flower가 안 들어가져요."**
    - 도커 컴포즈 파일에서 `flower` 부분의 포트 포워딩(`5555:5555`)이 되어 있는지, 
    - 그리고 위에서 말한 `--profile` 옵션을 줬는지 꼭 확인해.