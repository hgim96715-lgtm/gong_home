---
aliases:
  - crontab
  - 스케줄링
  - 자동화
  - 배치
  - cron
  - at
tags:
  - Linux
  - Airflow
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Shell_Script]]"
  - "[[Linux_Background_Jobs]]"
  - "[[Airflow_DAG_Skeleton]]"
---


# Linux_Scheduling_Crontab — 자동 실행 스케줄러

## 한 줄 요약

```
at     → 일회성 작업 예약 (한 번만 실행)
cron   → 반복 작업 예약 (주기적으로 실행)

at     = "내일 새벽 2시에 딱 한 번만 실행해줘"
crontab = "매일 새벽 2시에 실행해줘"
```

---

---

# ① at — 일회성 작업 예약 ⭐️

```
한 번만 실행하면 되는 작업에 사용
서버 자원 적은 새벽에 단발성 유지보수 스크립트 실행
```

## 설치 및 데몬 확인

```bash
sudo apt-get install -y at
sudo systemctl status atd      # atd 데몬 상태 확인
sudo systemctl start atd       # 실행 안 됐으면 시작
```

```
atd = at daemon
at 명령어가 동작하려면 atd 데몬이 실행 중이어야 함
cron 도 마찬가지: cron 데몬 실행 중이어야 crontab 동작
```

## 기본 사용법

```bash
# 1분 후 실행
at now + 1 minute
at> echo "job done" > ~/output.txt
at> <Ctrl-D>       # ← 입력 완료 후 Ctrl-D 로 등록
# job 1 at Sat Apr 18 18:27:00 2026

# 특정 시간에 실행
at 02:30            # 오늘 새벽 2시 30분
at 02:30 tomorrow   # 내일 새벽 2시 30분
at now + 2 hours    # 2시간 후
at now + 3 days     # 3일 후
```

```
at 시간 지정 형식:
  now + N minute / hour / day / week
  HH:MM                → 오늘 특정 시각
  HH:MM tomorrow       → 내일 특정 시각
  HH:MM YYYY-MM-DD     → 특정 날짜
```

## atq / atrm — 예약 작업 확인 / 취소

```bash
# 대기 중인 작업 목록
atq
# 2   Sat Apr 18 18:37:00 2026 a labex
# ↑   ↑
# Job ID  실행 예정 시각

# 특정 작업 취소
atrm 2         # Job ID 2 번 취소
atrm 2 3 4     # 여러 개 동시 취소

# 취소 후 확인
atq
```

```
실수 방지 패턴:
  atrm 전에 반드시 atq 로 번호 확인
  잘못된 번호 입력하면 다른 중요 작업이 삭제됨
```

---

---

# ② crontab vs at 비교 ⭐️

```
at:
  일회성 (딱 한 번)
  atd 데몬 필요
  atq 로 대기 목록 확인
  atrm 으로 취소

cron:
  반복 실행 (매분 / 매일 / 매주 ...)
  cron 데몬 필요
  crontab -l 로 목록 확인
  crontab -r 또는 편집으로 삭제

언제 뭘 쓰나:
  단발성 유지보수  → at
  정기 백업       → crontab
  정기 로그 삭제   → crontab
```

---

---

# ③ crontab 기본 명령어

```bash
crontab -e   # 스케줄 편집 (vi 에디터 열림)
crontab -l   # 등록된 스케줄 목록 확인
crontab -r   # 모든 스케줄 삭제 ⚠️ 묻지도 않고 전부 삭제
```

```bash
# 다른 유저의 crontab 확인 (관리자 권한)
sudo crontab -u username -e
```

## crontab -r 주의 ⭐️

```
-r 은 전체 삭제 / 확인 없음 / 되돌리기 불가

권장 방법:
  특정 작업만 끄고 싶으면:
  crontab -e 로 편집기 진입 → 해당 줄 앞에 # 주석 처리
  → 나중에 다시 필요하면 주석 해제

절대 피해야 할 상황:
  crontab -r 치려다 실수로 치는 경우
  → 중요 스케줄 전부 날아감
```

---

---

# ④ 문법 — 별 5개의 법칙

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

# ⑤ 예제 모음

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

# ⑥ 로그 남기기 — 리다이렉션 필수

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

# ⑦ 스케줄 삭제

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

## ⑤ at 실행이 안 돼요 — atd 데몬 확인

```bash
# atd 데몬 상태 확인
sudo systemctl status atd

# 실행 안 됐으면 시작
sudo systemctl start atd

# cron 데몬도 동일
sudo systemctl status cron
```

## ⑥ at 작업 번호 헷갈려서 잘못 취소했어요

```
atrm 전에 반드시 atq 로 번호 확인
여러 작업이 등록된 경우 번호 꼭 확인 후 삭제
```

```bash
atq              # 번호 확인
atrm 정확한번호  # 확인 후 삭제
```