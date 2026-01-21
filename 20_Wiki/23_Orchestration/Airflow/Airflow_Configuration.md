---
aliases:
  - Airflow 환경설정
  - load_examples
  - 예제 숨기기
  - airflow.cfg
tags:
  - Airflow
  - 설정
related:
  - "[[Airflow_Installation]]"
---
## 개념 한 줄 요약

Airflow 설치 시 기본적으로 제공되는 **'학습용 예제 DAG들(Tutorial DAGs)'을 로딩하지 않도록 막는 설정**입니다.
(스마트폰 처음 샀을 때 깔려있는 통신사 기본 앱을 삭제하는 것과 같습니다.)

---
## 왜 필요한가 (Why)

**문제 상황:**
`load_examples = True` (기본값) 상태로 실행하면 다음과 같은 문제가 생깁니다.
1.  **가시성 저하:** 내가 만든 소중한 DAG 1개를 찾기 위해, 쓸데없는 예제 DAG 30개 사이를 뒤져야 합니다. 
2.  **성능 낭비:** Airflow 스케줄러는 DAG 폴더에 있는 파일들을 끊임없이 읽어서 분석(Parsing)합니다. 안 쓰는 예제 파일 30개를 계속 읽느라 컴퓨터 자원(CPU)을 낭비하게 됩니다.

**해결책:**
`load_examples = False`로 설정하면 예제들이 싹 사라지고, **내가 만든 DAG만 깔끔하게** 화면에 남습니다. 

---
## 실무 맥락에서의 사용 이유

**프로덕션(실무) 환경에서는 무조건 `False`가 필수입니다.**
* 실수로 예제 DAG를 켜서(Trigger) 서버에 부하를 주는 것을 방지합니다.
* 해커가 예제 DAG의 취약점을 이용할 수도 있으므로 보안상 지우는 게 좋습니다.

---
## 코드 핵심 포인트 (설정 방법)

### Docker Compose 에서 수정

`docker-compose.yaml` 파일의 `x-airflow-common: &airflow-common` 섹션이나 `environment` 부분에 아래 한 줄을 추가/수정합니다.

```yaml
environment:
  # 코어 설정(AIRFLOW__CORE)의 예제 로드(LOAD_EXAMPLES)를 끈다(False)
  AIRFLOW__CORE__LOAD_EXAMPLES: 'false'
```

### airflow.cfg 파일 직접 수정 시 (Standalone)

`airflow.cfg` 파일을 열어서 아래 항목을 찾아 수정합니다.

```Ini
[core]
load_examples = False
```

---
## 초보자가 자주 착각하는 포인트

1. **"설정을 바꿨는데 화면에 그대로 있어요!"**
    - 설정을 바꾸고 나서 **Docker를 재시작(`docker compose down` -> `up`)** 해야 적용됩니다.
    - 그래도 남아있다면, 이미 DB에 기록이 남아서 그렇습니다. `docker compose down -v` (볼륨 삭제 옵션)로 DB까지 싹 밀고 다시 켜면 깨끗해집니다. (주의: 기존 데이터 다 날아감)
        
2. **"예제를 보고 공부하고 싶은데요?"**
    - 그럴 때는 `True`로 두고 보셔도 됩니다. 
    - 하지만 공부가 끝나고 내 프로젝트를 시작할 때는 끄는 것을 권장합니다.