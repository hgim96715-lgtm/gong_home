---
aliases:
  - tar
  - 압축
  - 백업
  - gzip
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_File_Commands]]"
  - "[[Linux_Superuser]]"
---

# Linux_Archive — 압축 & 백업

## 한 줄 요약

```
tar   = 여러 파일을 하나로 묶기 (아카이브)
gzip  = 파일 압축 (용량 줄이기)
실무: tar + gzip 같이 써서 묶기 + 압축 동시에
```

---

---

# ① tar — 파일 묶기 ⭐️

```
tar = Tape ARchiver
  여러 파일 / 디렉토리를 하나의 .tar 파일로 묶음
  압축은 별도 (gzip 과 함께 쓰면 .tar.gz)
```

## 기본 옵션

|옵션|의미|
|---|---|
|`-c`|Create — 새 아카이브 생성|
|`-x`|eXtract — 추출 (압축 해제)|
|`-t`|lisT — 내용 목록 보기|
|`-v`|Verbose — 처리 파일 목록 출력|
|`-f`|File — 아카이브 파일명 지정 (항상 마지막)|
|`-z`|gzip 압축 (.tar.gz)|
|`-j`|bzip2 압축 (.tar.bz2)|

## 핵심 패턴 3개

```bash
# 생성 (create)
tar -cvf archive.tar /path/to/dir

# 추출 (extract)
tar -xvf archive.tar

# 목록 확인 (list)
tar -tvf archive.tar
```

```
c 로 만들고 → x 로 풀기
한 글자만 바꾸면 됨
```

---

---

# ② 백업 생성 — sudo tar ⭐️

## 기본 백업

```bash
# /home 전체 백업
sudo tar -cvf ~/backup.tar /home

# 실행 시 나오는 메시지:
# tar: Removing leading '/' from member names  ← 정상! 에러 아님
```

```
"Removing leading '/'" 메시지 이유:
  /home → home/ 으로 절대경로를 상대경로로 변환해서 저장
  왜?: 나중에 압축 풀 때 기존 /home 을 덮어쓰는 대참사 방지
  /tmp 같은 다른 곳에 풀면 home/labex/... 구조로 안전하게 생성
```

## sudo 로 만든 파일 소유권 주의

```bash
sudo tar -cvf ~/backup.tar /home

ls -l ~/backup.tar
# -rw-r--r-- 1 root root 10240 ... backup.tar
#              ↑ 소유자가 root!

# sudo 로 만들면 파일 소유자 = root
# 나중에 이동/삭제 시 다시 sudo 필요
```

## 날짜 포함 백업 파일명 패턴

```bash
# 날짜를 파일명에 포함
sudo tar -cvf ~/backup_$(date +%Y%m%d).tar /home
# → backup_20260415.tar

# 시각까지
sudo tar -cvf ~/backup_$(date +%Y%m%d_%H%M).tar /home
# → backup_20260415_1430.tar
```

---

---

# ③ 복구 — tar 추출 ⭐️

## /tmp 에서 복구 (실무 권장 패턴)

```bash
# 1. /tmp 로 이동 (안전한 임시 공간)
cd /tmp

# 2. 추출 (sudo 불필요 — /tmp 는 모두 쓰기 가능)
tar -xvf ~/backup.tar

# 3. 확인
ls /tmp/home/labex/.bashrc
```

```
왜 /tmp 에서 추출하나:
  기존 /home 을 덮어쓰면 데이터 꼬일 위험
  /tmp 에 풀면 home/labex/... 구조로 안전하게 복구
  필요한 파일만 꺼내서 원위치로 복사

/tmp 특징:
  모든 사용자 읽기/쓰기 가능 (권한 777)
  재부팅 시 자동 삭제
  임시 복구 테스트에 최적
```

## 특정 파일만 추출

```bash
# 특정 파일만 추출
tar -xvf backup.tar home/labex/.bashrc

# 특정 디렉토리만 추출
tar -xvf backup.tar home/labex/

# 다른 경로에 추출
tar -xvf backup.tar -C /tmp/restore/
```

---

---

# ④ gzip 압축 — .tar.gz ⭐️

```
tar  = 묶기만 (용량 그대로)
gzip = 압축 (용량 줄임)
-z 옵션 = tar + gzip 동시에
```

## 압축 아카이브 생성

```bash
# .tar.gz 생성 (-z 추가)
sudo tar -czvf backup.tar.gz /home

# 압축률 비교
ls -lh backup.tar backup.tar.gz
```

## 압축 해제

```bash
tar -xzvf backup.tar.gz
tar -xzvf backup.tar.gz -C /tmp/
```

## 확장자 종류

|확장자|옵션|특징|
|---|---|---|
|`.tar`|(없음)|묶기만 / 압축 없음|
|`.tar.gz` / `.tgz`|`-z`|gzip 압축 / 빠름|
|`.tar.bz2`|`-j`|bzip2 압축 / 더 작지만 느림|
|`.tar.xz`|`-J`|xz 압축 / 가장 작지만 가장 느림|

---

---

# ⑤ gzip / bzip2 / zip — 단일 파일 압축

## gzip

```bash
gzip file.txt          # file.txt → file.txt.gz (원본 삭제)
gzip -k file.txt       # 원본 유지
gzip -d file.txt.gz    # 압축 해제
gunzip file.txt.gz     # 압축 해제 (동일)
gzip -l file.txt.gz    # 압축률 확인
```

## bzip2

```bash
bzip2 file.txt         # file.txt → file.txt.bz2
bzip2 -d file.txt.bz2  # 압축 해제
bunzip2 file.txt.bz2   # 압축 해제 (동일)
```

## zip / unzip (Windows 호환)

```bash
zip archive.zip file1 file2   # 파일 압축
zip -r archive.zip mydir/     # 디렉토리 압축
unzip archive.zip             # 압축 해제
unzip archive.zip -d /tmp/    # 특정 경로에 해제
unzip -l archive.zip          # 내용 목록 확인
```

---

---

# ⑥ 실무 패턴

## 백업 + 복구 전체 흐름

```bash
# 1. 백업 생성
sudo tar -czvf ~/backup_$(date +%Y%m%d).tar.gz /home

# 2. 백업 파일 확인
ls -lh ~/backup_*.tar.gz

# 3. 내용 확인 (열지 않고)
tar -tzvf ~/backup_20260415.tar.gz | head -20

# 4. /tmp 에서 복구 테스트
cd /tmp
tar -xzvf ~/backup_20260415.tar.gz

# 5. 특정 파일만 원위치로 복사
cp /tmp/home/labex/.bashrc ~/.bashrc
```

## 로그 디렉토리 백업 (데이터 엔지니어 실무)

```bash
# Airflow 로그 백업
sudo tar -czvf /backup/airflow_logs_$(date +%Y%m%d).tar.gz /opt/airflow/logs/

# 오래된 백업 자동 삭제 (30일 이상)
find /backup/ -name "*.tar.gz" -mtime +30 -delete
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`Removing leading '/'` 보고 놀람|정상 메시지|무시해도 됨 (안전 메커니즘)|
|풀었더니 기존 파일 덮어씀|원위치에서 추출|`/tmp` 에서 추출 후 필요 파일만 복사|
|sudo 로 만든 백업파일 수정 못함|소유자 root|`sudo` 붙여서 작업|
|`-f` 옵션 마지막에 안 씀|파일명이 뒤에 와야 함|`-cvf 파일명` 순서 주의|
|압축 해제 후 파일 없음|경로 확인 안 함|`tar -tvf` 로 내용 먼저 확인|