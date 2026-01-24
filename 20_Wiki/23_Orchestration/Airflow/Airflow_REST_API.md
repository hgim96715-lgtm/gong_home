---
aliases:
  - Airflow API
  - REST API
  - Trigger DAG via API
  - Stable API
tags:
  - Airflow
  - API
related:
  - "[[Python_Requests_Response]]"
  - "[[HTTP_Status_Codes]]"
---
## 개념 한 줄 요약

**Airflow REST API**는 사람이 UI 버튼을 누르는 대신, **코드(Script)나 외부 프로그램이 Airflow에게 명령을 내릴 수 있게 해주는 인터페이스**야.

>[전체 API 레퍼런스 메인](https://airflow.apache.org/docs/apache-airflow/stable/stable-rest-api-ref.html)

>http://localhost:8080/api/v2/dags

----
## 왜 필요한가 (Why)

 **문제점:**
 -  "매일 9시" 같은 정기 스케줄 말고, "회사 웹사이트에 사용자가 파일을 업로드하면 즉시 분석 DAG를 돌려줘!" 같은 요구사항이 들어오면? 크론(Cron) 스케줄로는 불가능해.
 
**해결책:**
- 웹사이트 백엔드 서버가 Airflow API한테 **"야, 이 DAG 지금 당장 실행해(Trigger)!"** 라고 신호를 보내면 해결됨. 즉, **이벤트 기반(Event-driven) 파이프라인**을 만들 수 있어.

---
## Practical Context (실무 활용)

현업에서는 주로 이럴 때 씁니다.

1.  **External Trigger:** AWS Lambda, Slack 챗봇, 혹은 사내 어드민 페이지에서 버튼을 누르면 DAG가 돌게 할 때.
2.  **CI/CD Integration:** 깃허브(GitHub)에 코드가 머지되면 자동으로 테스트 DAG를 실행시킬 때.
3.  **Monitoring:** 사내 대시보드에 Airflow 상태를 띄우고 싶을 때 (API로 상태값만 긁어감).

---
## Code Core Points

API를 쓰려면 `airflow.cfg` 설정이 먼저 되어 있어야 하고, **Basic Auth** (ID/PW) 인증을 거쳐야 해.

### 1) 사전 설정 (`airflow.cfg`)

API는 보안 때문에 기본적으로 막혀있거나 인증 방식이 다를 수 있어.

```ini
[api]
auth_backends = airflow.providers.fab.auth_manager.api.auth.backend.session
```

### 2) Python `requests`로 DAG 실행하기 (Trigger DAG)

우리가 배웠던 `requests`를 여기서 써먹는 거야!

```python
import requests
from requests.auth import HTTPBasicAuth

# 1. 설정
AIRFLOW_URL = "http://localhost:8080/api/v2"
DAG_ID = "my_target_dag"
auth = HTTPBasicAuth("admin", "admin") # Airflow 로그인 정보

# 2. 요청 보내기 (POST: 무언가 생성/실행할 땐 POST!)
endpoint = f"{AIRFLOW_URL}/dags/{DAG_ID}/dagRuns"

# conf를 통해 데이터를 같이 던져줄 수 있음 (매우 유용!)
payload = {
    "conf": {"filename": "sales_data_2024.csv"} 
}

response = requests.post(endpoint, json=payload, auth=auth)

# 3. 응답 확인 (아까 배운 Status Code!)
if response.status_code == 200:
    print("🚀 DAG 실행 성공!")
    print(response.json())
elif response.status_code == 401:
    print("⛔ 인증 실패: 아이디/비번 확인하세요.")
elif response.status_code == 404:
    print("❌ DAG ID 틀림: 그런 DAG 없는데요?")
else:
    print(f"💥 에러 발생: {response.status_code}")
    print(response.text)
```

---
## 코드 분석

API를 쓸 때 가장 많이 쓰는 엔드포인트(주소) 3대장이야.

1. **POST `/dags/{dag_id}/dagRuns`**: 가장 중요! 특정 DAG를 즉시 실행시킴.
	- `conf` 파라미터로 데이터를 넘겨줄 수 있다는 게 핵심.

2. **GET `/dags`**: 현재 등록된 모든 DAG 목록을 가져옴.
    
3. **GET `/dags/{dag_id}/dagRuns/{dag_run_id}`**: 실행시킨 DAG가 성공했는지 실패했는지 상태를 확인.

---
## 초보자가 자주 착각하는 포인트

1. **"Experimental API 쓰면 되나요?"**
    - 절대 안 돼! 
    - 옛날 인터넷 글 보면 `experimental` API를 쓰라고 되어 있는데, 그건 Airflow 2.0부터 버려졌어(Deprecated). 
    - 무조건 **"Stable API (v1)"** 를 써야 해.
        
2. **"CORS 에러가나요."**
    - 웹 브라우저(프론트엔드)에서 자바스크립트로 직접 Airflow API를 호출하면 보안 때문에 막혀. 
    - 보통은 백엔드 서버(Python, Java)를 거쳐서 호출해야 해.

3. **"설정(`conf`) 보냈는데 못 받아요."**
    - API로 보낸 데이터는 DAG 안에서 `{{ dag_run.conf['key'] }}` 형태로 꺼내 써야 해. 
    - 일반 변수랑 꺼내는 법이 달라.
 