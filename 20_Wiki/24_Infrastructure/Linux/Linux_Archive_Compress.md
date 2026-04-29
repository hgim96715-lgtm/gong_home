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

# 특정 디렉토리에 해제
tar -xzf archive.tar.gz -C /tmp/

# 과정 출력하며 해제
tar -xzvf archive.tar.gz
```

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

| 실수               | 원인                | 해결                           |
| ---------------- | ----------------- | ---------------------------- |
| 압축 후 원본 삭제 안 함   | 순서 잊음             | `tar` → `tar -tzf` 확인 → `rm` |
| `rm *` 잘못 사용     | 와일드카드 범위 착각       | 삭제 전 `ls *_2023-*.log` 로 확인  |
| `-f` 옵션 순서 틀림    | `-fczf` 처럼 f 앞에 씀 | `-czf` 순서 유지 (f 는 항상 마지막)    |
| 압축 해제 후 파일 위치 모름 | 경로 지정 안 함         | `-C 경로` 로 명시적 지정             |