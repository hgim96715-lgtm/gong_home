---
aliases:
  - tar
  - 압축
  - 아카이브
  - 로그 로테이션
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_File_Move_Copy]]"
  - "[[Linux_File_Delete]]"
  - "[[Linux_Background_Jobs]]"
  - "[[Shell_Cron_Job]]"
---
# Linux_Archive_Compress — 압축 & 아카이브

## 한 줄 요약

```
tar    = 여러 파일을 하나로 묶기
gzip   = 파일 크기 줄이기
tar.gz = 묶기 + 압축 동시에 (가장 많이 씀)
```

---

---

# ① tar 핵심 옵션

```bash
tar [옵션] 아카이브명 파일들

옵션:
  c  = Create  (새 아카이브 생성)
  x  = eXtract (압축 해제)
  t  = lisT    (목록만 보기)
  z  = gZip    (.tar.gz 형식)
  f  = File    (파일명 지정, 항상 마지막)
  v  = Verbose (처리 과정 출력)
```

---

---

# ② 압축 생성 — tar -czf ⭐️

```bash
# 기본 패턴: tar -czf [아카이브명] [압축할 파일/폴더]
tar -czf archive.tar.gz 파일1 파일2

# 와일드카드로 특정 패턴 파일만
tar -czf old_logs.tar.gz *_2023-*.log
# *_2023-*.log = 이름에 _2023- 포함된 .log 파일 전부

# 폴더 전체 압축
tar -czf backup.tar.gz src/
tar -czf project.tar.gz ~/project/

# 날짜 포함 백업
tar -czf backup_$(date +%Y%m%d).tar.gz /etc/nginx/
# → backup_20260420.tar.gz
```

## -czvf 각 옵션 상세 ⭐️

```
c  (Create)  = 새 아카이브 파일 생성
z  (Zip)     = gzip 으로 압축 (.tar.gz 형식)
v  (Verbose) = 처리 중인 파일명을 화면에 출력
f  (File)    = 저장할 아카이브 파일명 지정 → 반드시 마지막!

tar -czvf archive.tar.gz data/
     ↑↑↑↑ ↑             ↑
     옵션  아카이브이름   압축대상
```

```
v 옵션:
  있으면 → 압축 중인 파일들이 화면에 쭉 출력됨
  없으면 → 조용히 압축만 진행

  무거운 작업(수백 파일) 걸어둘 때:
    v 있으면 진행 상황 눈으로 확인 가능
    cron 자동화 시에는 v 빼고 로그 파일로 리다이렉션

  nohup tar -czvf backup.tar.gz data/ > backup.log 2>&1 &
  tail -f backup.log   ← 실시간 진행 확인
```

```
f 옵션 위치가 왜 마지막인가:
  f 는 "다음 인자가 파일명" 이라는 의미
  f 뒤에 아카이브 이름이 바로 와야 함

  ✅ tar -czvf archive.tar.gz data/
               ↑ f 다음 = 파일명

  ❌ tar -fczv archive.tar.gz data/
         ↑ f 가 앞에 오면 의미 혼동

  현장 실수 패턴:
  tar -czvf 파일들 archive.tar.gz  ← 순서 바꾸면 에러
  tar -czvf archive.tar.gz 파일들  ← 올바른 순서
```

## -czf vs -czvf 선택 기준

```
-czf  (v 없음)  조용히 압축 / cron 자동화 / 빠른 실행
-czvf (v 있음)  진행 상황 확인 / 수동 실행 / 디버깅

실무 패턴:
  수동으로 큰 폴더 압축:    tar -czvf → 진행 상황 보기
  cron 자동화 스크립트:     tar -czf  >> log.txt 2>&1
```

```
-czf 옵션 읽는 법:
  c = 생성 (create)
  z = gzip 압축
  f = 파일명 (다음에 아카이브 이름 옴)

  -czf old_logs.tar.gz *_2023-*.log
       ↑ 아카이브 이름  ↑ 압축할 대상
```

---

---

# ③ 목록 확인 — tar -tzf ⭐️

```bash
# 압축 풀지 않고 내용만 확인
tar -tzf old_logs.tar.gz
# app_2023-01-01.log
# app_2023-01-02.log
# error_2023-01-01.log
# ...

# 상세 정보 포함 (-v 추가)
tar -tzvf archive.tar.gz
```

```
압축 풀기 전에 반드시 확인하는 습관:
  1. tar -tzf 로 목록 확인
  2. 예상한 파일이 맞는지 검증
  3. 그 다음 압축 해제
```

---

---

# ④ 압축 해제 — tar -xzf

```bash
# 현재 디렉토리에 해제
tar -xzf archive.tar.gz

# 특정 디렉토리에 해제 (-C)
tar -xzf archive.tar.gz -C /tmp/

# 과정 출력하며 해제
tar -xzvf archive.tar.gz
```

## 부분 복원 — 특정 파일만 ⭐️

```bash
# 아카이브 전체 해제 대신 파일 하나만 복원
tar -xzvf backups/system-backup.tar.gz config/app.conf
#                                       ↑ 복원할 파일 경로 명시

# 복원 확인
ls -l config/app.conf
cat config/app.conf
```

```
장애 상황에서 수십 GB 전체 백업을 풀면 시간 낭비
→ 파일명 명시 → 해당 파일만 즉시 복원
→ 서비스 다운타임 최소화
```

## -T 옵션 — 목록 파일로 백업 ⭐️

```bash
# backup-list.txt 예시
# data/
# config/
# logs/

# -T 로 목록 파일 읽어서 백업
tar -czvf backups/system-backup.tar.gz -T backup-list.txt
#                                       ↑ 경로 목록 파일

# 백업 무결성 확인
tar -tzvf backups/system-backup.tar.gz > backup-contents.txt
cat backup-contents.txt
```

```
왜 목록 파일을 쓰나:
  명령줄에 경로 20개 나열 → 오타 유발
  backup-list.txt 한 번 만들어두면 반복 사용
  수정도 파일 하나만 편집 → 관리 편함
```

## -C 옵션 — 기준 디렉토리 지정

```bash
# -C 경로 = 해당 경로 기준으로 상대 경로 압축
tar -czf backup.tar.gz -C /home/labex/project data config logs
#                       ↑ 이 경로에서        ↑ 이 폴더들 압축

# 해제 시 -C 로 위치 지정
tar -xzf backup.tar.gz -C /restore/location/

# vs 경로 통째로 (차이)
tar -czf backup.tar.gz /home/labex/project/data
# → 아카이브 안 경로: home/labex/project/data/...

tar -czf backup.tar.gz -C /home/labex/project data
# → 아카이브 안 경로: data/...  (깔끔)
```

## cron 자동 백업 — 날짜 타임스탬프 패턴 ⭐️

```bash
# 매 분마다 날짜/시간 포함 백업 파일 생성
* * * * * tar -czf /home/labex/project/backups/backup-$(date +\%Y-\%m-\%d_\%H-\%M-\%S).tar.gz -C /home/labex/project data config logs

# cron 에서 % 는 줄바꿈으로 인식 → \% 로 이스케이프 필수!
# 터미널에서 직접 실행할 때는 % 그대로
tar -czf backup_$(date +%Y-%m-%d_%H-%M-%S).tar.gz -C /home/labex/project data config logs
```

```
cron % 이스케이프:
  터미널:  date +%Y-%m-%d    (% 그대로)
  cron:    date +\%Y-\%m-\%d (\% 이스케이프)

날짜 포맷:
  %Y = 연도 (2026)
  %m = 월   (04)
  %d = 일   (20)
  %H = 시   (10)
  %M = 분   (30)
  %S = 초   (00)

  결과: backup-2026-04-20_10-30-00.tar.gz
  → 같은 이름으로 덮어쓰기 방지
```

>[[Shell_Cron_Job]] 참고 

---

---

# ⑤ 와일드카드 패턴 — rm 과 조합 ⭐️

```bash
# 특정 패턴 파일만 압축 후 삭제 (로그 정리)
cd ~/project/logs

# 1. 2023년 로그 압축
tar -czf old_logs.tar.gz *_2023-*.log

# 2. 압축 확인
tar -tzf old_logs.tar.gz

# 3. 원본 삭제
rm *_2023-*.log

# 4. 결과 확인
ls
```

```
와일드카드 패턴:
  *        = 모든 문자 (아무거나)
  ?        = 한 문자
  [abc]    = a, b, c 중 하나

*_2023-*.log 해석:
  *        = 아무 이름
  _2023-   = 고정 문자
  *        = 아무 이름
  .log     = .log 확장자

예시 매칭:
  app_2023-01-01.log   ✅
  error_2023-12-31.log ✅
  app_2024-01-01.log   ❌ (2024는 해당 없음)
```

---

---

# ⑥ 로그 로테이션 패턴 ⭐️

```
문제:
  서버 장기 운영 → 로그 파일이 GB 단위로 쌓임
  디스크 100% → 시스템 장애

해결:
  오래된 로그 → 압축 보관 → 원본 삭제
  이것이 Log Rotation 의 핵심 원리
```

```bash
#!/bin/bash
# 로그 로테이션 스크립트 예시

LOG_DIR="/var/log/myapp"
ARCHIVE_DIR="/backup/logs"
DAYS_TO_KEEP=30

# 30일 이상 된 로그 압축
find $LOG_DIR -name "*.log" -mtime +$DAYS_TO_KEEP | while read f; do
    tar -czf "$ARCHIVE_DIR/$(basename $f).tar.gz" "$f"
    rm "$f"
done

echo "로그 정리 완료: $(date)"
```

---

---

# ⑦ 형식별 비교

|확장자|옵션|특징|
|---|---|---|
|`.tar`|`-cf` / `-xf`|묶기만 (압축 없음)|
|`.tar.gz` / `.tgz`|`-czf` / `-xzf`|gzip 압축 (빠름)|
|`.tar.bz2`|`-cjf` / `-xjf`|bzip2 (더 작지만 느림)|
|`.tar.xz`|`-cJf` / `-xJf`|xz 압축 (가장 작음)|

---

---

# 명령어 한눈에

|명령어|역할|
|---|---|
|`tar -czf 결과.tar.gz 파일들`|압축 생성|
|`tar -tzf 파일.tar.gz`|내용 목록 확인|
|`tar -xzf 파일.tar.gz`|압축 해제 (현재 위치)|
|`tar -xzf 파일.tar.gz -C 경로`|특정 경로에 해제|
|`rm *.log`|패턴 파일 삭제|
|`rm *_2023-*.log`|특정 패턴만 삭제|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|압축 후 원본 삭제 안 함|순서 잊음|`tar` → `tar -tzf` 확인 → `rm`|
|`rm *` 잘못 사용|와일드카드 범위 착각|삭제 전 `ls *_2023-*.log` 로 확인|
|`-f` 옵션 순서 틀림|`-fczf` 처럼 f 앞에 씀|`-czf` 순서 유지 (f 는 항상 마지막)|
|압축 해제 후 파일 위치 모름|경로 지정 안 함|`-C 경로` 로 명시적 지정|