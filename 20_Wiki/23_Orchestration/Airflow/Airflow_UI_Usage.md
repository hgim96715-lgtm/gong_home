---
aliases:
  - Airflow UI 사용법
  - 로그 확인하는 법
  - 재시도 방법
  - Grid View
tags:
  - Airflow
related:
  - "[[00_Airflow_HomePage]]"
  - "[[Airflow_DAG_Concept]]"
  - "[[Airflow_Architecture]]"
---
# Airflow_UI_Usage — Airflow UI 사용법

## 한 줄 요약

```
DAG 시각화 / 실행 상태 모니터링 / 로그 확인 / 재시도
통합 관제 센터
```

---

---

# ① 주요 화면

## Graph View — 의존성 구조 확인

```
Task 간 선후 관계를 그래프로 표현
"Extract 다음에 Transform 이 돌고, 그다음 Load" 한눈에 파악
현재 실행 중인 Task 가 어디인지 실시간 확인
```

## Grid View — 실행 이력 확인

```
날짜(열) × Task(행) 의 바둑판 형태
색으로 상태 표시:
  초록색  Success
  빨간색  Failed
  연두색  Running
  회색    Skipped

"지난주 화요일에 왜 실패했지?" 패턴 분석 시 유용
```

---

---

# ② 자주 쓰는 조작

## 로그 확인 — 가장 중요 ⭐️

```
Worker 컴퓨터 직접 접속 없이 웹에서 바로 로그 확인 가능
```

```
① Grid View 클릭
② 보고 싶은 날짜(열) + Task(행) 이 만나는 네모 클릭
③ 오른쪽 패널 → Logs 탭 클릭
```

```
로그에서 확인할 것:
  에러 메시지
  실행 시간 (언제 시작 / 언제 끝)
  print() 출력 내용
  어떤 파라미터로 실행됐는지
```

## 실패한 Task 재시도 — Clear

```
에러 고치고 실패한 부분부터 다시 돌릴 때

① 실패한 빨간 네모(Failed Task Instance) 클릭
② 오른쪽 패널 → Clear 버튼 클릭
③ Scheduler 가 해당 Task 를 다시 실행

Clear = "이 Task Instance 상태를 초기화해서 다시 돌려줘"
전체 DAG 를 처음부터 안 돌려도 됨 → 실패한 Task 만 재실행
```

## 수동 실행 — Trigger

```
스케줄 시간 외에 지금 당장 실행하고 싶을 때

① DAG 목록에서 오른쪽 상단 ▶ (재생 버튼) 클릭
② Trigger DAG 선택
③ 필요하면 파라미터 입력 (Conf 항목)
```

## DAG ON/OFF — 토글 스위치

```
DAG 목록 왼쪽의 토글 스위치
OFF: 스케줄 일시 중지 (이미 실행 중인 Task 는 계속)
ON: 스케줄 재개

개발 중이거나 문제 있을 때 잠깐 끄는 용도
```

---

---

# ③ Task Instance 상태

|색상|상태|의미|
|---|---|---|
|초록|Success|성공|
|빨강|Failed|실패|
|연두|Running|실행 중|
|노랑|Queued|대기 중|
|주황|Up for Retry|재시도 예정|
|회색|Skipped|건너뜀|
|보라|Upstream Failed|앞 Task 실패로 미실행|

---

---

# ④ 트러블슈팅 — UI 로 확인하는 법

```
"작업이 안 돌아요" → Grid View 에서 상태 확인
  빨간색: 로그 확인 → 에러 메시지 파악
  노란색: Queue 에 쌓임 → Worker 상태 확인
  보라색: 앞 Task 실패 → Upstream 확인

"어제는 됐는데 오늘 안 돼요" → Grid View 날짜 비교
  어제 Task Instance 로그 vs 오늘 Task Instance 로그 비교

"코드 고쳤는데 반영 안 돼요" → DAG 파일 재파싱 대기
  Scheduler 가 DAG 파일 주기적으로 읽음 (30초 정도 걸릴 수 있음)
  Code 탭에서 현재 파싱된 코드 확인 가능
```

---

---

# 면접 답변

```
"Airflow UI 로 주로 뭘 하셨나요?"

"Graph View 로 DAG 의존성 구조를 파악하고
 Grid View 로 실행 이력을 모니터링했습니다.
 실패한 Task Instance 의 로그를 분석해서 원인 파악 후
 Clear 로 해당 부분만 재실행하는 방식으로 트러블슈팅했습니다."
```