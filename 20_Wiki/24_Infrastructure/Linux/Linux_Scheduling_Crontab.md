---
aliases:
  - crontab
  - 스케줄링
  - 자동화
  - 배치
  - cron
tags:
  - Linux
  - Airflow
related:
  - "[[Airflow_Scheduling]]"
  - "[[Time_Synchronization]]"
  - "[[Process_Management]]"
  - "[[Airflow_DAG_Skeleton]]"
  - "[[00_Linux_HomePage]]"
---


# Linux_Scheduling_Crontab — 자동 실행 스케줄러

## 한 줄 요약

```
리눅스의 알람 시계
내가 자는 동안에도 정해진 시간에 명령어를 자동 실행

Cron    → 백그라운드에서 예정된 작업을 실행하는 데몬(서비스)
Crontab → Cron 에게 "언제 무엇을 할지" 적어주는 시간표 + 명령어
```

---

---

# ① 기본 명령어 3개

```bash
crontab -e   # 스케줄 편집 (vi 에디터 열림)
crontab -l   # 등록된 스케줄 목록 확인
crontab -r   # 모든 스케줄 삭제 ⚠️ 묻지도 않고 전부 삭제
```

```bash
# 다른 유저의 crontab 확인 (관리자 권한)
sudo crontab -u username -e
```

---

---

# ② 문법 — 별 5개의 법칙

```
분 시 일 월 요일  실행할_명령어

  *  *  *  *  *  command
  ┬  ┬  ┬  ┬  ┬
  │  │  │  │  └─ 요일 (0~7, 0·7=일 / 1=월 / ... / 6=토)
  │  │  │  └──── 월   (1~12)
  │  │  └──────── 일   (1~31)
  │  └──────────── 시   (0~23)
  └────────────────  분   (0~59)

암기: 분-시-일-월-요
```

## 특수 기호

|기호|의미|예시|
|---|---|---|
|`*`|매번 (Every)|`* * * * *` = 매분|
|`/`|간격 (Step)|`*/10` = 10분마다|
|`,`|목록 (List)|`1,15` = 1일과 15일|
|`-`|범위 (Range)|`1-5` = 월~금|

---

---

# ③ 예제 모음

```bash
# 매분 실행 (테스트용)
* * * * * echo "every minute"

# 매일 새벽 02:30 (가장 흔한 배치 작업)
30 2 * * * /path/to/backup.sh

# 매일 오전 09:00
0 9 * * * /path/to/morning.sh

# 매주 월요일 오전 09:00
0 9 * * 1 /path/to/weekly_report.sh

# 매주 일요일 자정
0 0 * * 0 /path/to/weekly_backup.sh

# 10분마다
*/10 * * * * /path/to/check.sh

# 월~금, 9시~17시 매 정각
0 9-17 * * 1-5 /path/to/work_task.sh
```

## Airflow 에서 쓰는 cron 표현식

```python
schedule = '0 2 * * *'    # 매일 새벽 2시
schedule = '0 9 * * 1'    # 매주 월요일 9시
schedule = '@daily'        # 매일 자정 (= '0 0 * * *')
schedule = '@hourly'       # 매시 정각 (= '0 * * * *')
```

```
Airflow 의 schedule 이 바로 cron 문법
crontab 을 이해하면 Airflow 스케줄링도 그대로 적용
```

---

---

# ④ 로그 남기기 — 리다이렉션 필수

```
crontab 은 기본적으로 실행 결과를 화면에 안 보여줌
→ 리다이렉션(>>) 으로 파일에 저장해야 나중에 확인 가능
```

```bash
# 표준 출력 + 에러 모두 로그 파일에 저장
* * * * * /home/user/script.sh >> /home/user/cron.log 2>&1

# 날짜 포함해서 로그 남기기
* * * * * echo "$(date): job done" >> /home/user/cron.log

# 로그 실시간 확인
tail -f /home/user/cron.log   # Ctrl+C 로 중단
```

```
2>&1 의 의미:
  2  → 표준 에러 (stderr)
  >&1 → 표준 출력(stdout) 과 같은 곳으로
  → 에러와 일반 출력을 한 파일에 같이 저장
```

---

---

# ⑤ 스케줄 삭제

```bash
# 방법 A: 편집기로 들어가서 해당 줄만 삭제 (권장)
crontab -e
# → 해당 줄에서 dd (vi) 또는 Ctrl+K (nano) 로 삭제
# → 저장 후 종료

# 방법 B: 전체 삭제 (주의!)
crontab -r   # ⚠️ 확인 없이 전부 삭제
```

---

---

# 자주 하는 실수

## ① 스크립트가 안 돌아가요 — 절대경로 필수

```
crontab 은 환경변수($PATH) 를 모름
'python script.py' 라고 쓰면 python 이 어디 있는지 못 찾음
→ 무조건 절대경로 사용
```

```bash
# ❌ 상대경로 → 실행 안 됨
* * * * * python script.py

# ✅ 절대경로
* * * * * /usr/bin/python3 /home/user/script.py
* * * * * /bin/bash /home/user/script.sh

# python 위치 확인
which python3   # /usr/bin/python3
```

## ② 로그가 안 보여요

```
crontab 출력은 기본적으로 화면에 안 나옴
>> 리다이렉션으로 파일에 저장 필수 (④ 참고)
```

## ③ Permission denied

```
.sh 파일에 실행 권한 없으면 crontab 이 실행 못 함
→ chmod +x 로 권한 먼저 부여
```

```bash
chmod +x /path/to/script.sh
```

## ④ cron 표현식이 맞는지 확인하고 싶어요

```
https://crontab.guru
내가 짠 cron 표현식이 언제 실행되는지 해석해주는 사이트
등록 전에 항상 여기서 확인하는 습관
```