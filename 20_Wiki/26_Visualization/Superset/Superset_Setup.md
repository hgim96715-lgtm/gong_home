---
aliases:
  - Superset 초기화
  - Superset 설치
  - superset db upgrade
  - superset init
  - create-admin
tags:
  - Superset
related:
  - "[[00_Superset_HomePage]]"
  - "[[Superset_Core_Concepts]]"
  - "[[Superset_Database_Connection]]"
---

# Superset_Setup — 설치 및 초기화

## 한 줄 요약

```
컨테이너(또는 pip)로 Superset 을 올린 뒤
db upgrade → create-admin → init 순서로 최초 1회 초기화 필요
```

---

---

# ① 초기화가 필요한 이유

```
Superset 은 자체적으로 메타데이터 DB 를 가짐
  차트 설정 / 대시보드 레이아웃 / 사용자 정보 / 연결 정보 등을 저장

컨테이너(또는 pip install)만으로는 이 메타데이터 DB 가 비어있음
→ db upgrade  로 테이블 구조 생성
→ create-admin 으로 첫 관리자 계정 생성
→ init         으로 기본 역할/권한 세팅
```

```
설치 방식이 달라도 명령어는 동일
  pip 설치  → 터미널에서 직접 실행
  Docker   → docker exec 로 컨테이너 안에서 실행
```

---

---

# ② 초기화 순서 — pip 설치

```bash
# pip 로 설치한 경우 터미널에서 직접 실행
# 순서 반드시 지킬 것

# 1. 메타데이터 DB 테이블 생성
superset db upgrade

# 2. 관리자 계정 생성
superset fab create-admin \
  --username admin \
  --firstname Admin \
  --lastname Admin \
  --email admin@localhost \
  --password admin

# 3. 기본 역할 / 권한 초기화
superset init
```

---

---

# ③ 초기화 순서 — Docker

```bash
# Docker 로 실행한 경우
# docker exec 로 컨테이너 안에서 같은 명령어 실행

# 1. 메타데이터 DB 테이블 생성
docker exec -it train-superset superset db upgrade

# 2. 관리자 계정 생성
docker exec -it train-superset superset fab create-admin \
  --username admin \
  --firstname Admin \
  --lastname Admin \
  --email admin@localhost \
  --password admin

# 3. 기본 역할 / 권한 초기화
docker exec -it train-superset superset init
```

```
pip 설치 vs Docker 비교:
  명령어 자체는 완전히 동일
  Docker 는 앞에 docker exec -it {컨테이너명} 만 붙이면 됨
```

---

---

# ④ 각 명령어 역할

```
superset db upgrade
  Superset 내부 메타데이터 DB(SQLite 또는 PostgreSQL) 에
  차트 / 대시보드 / 유저 / 연결 정보 등을 저장할 테이블을 생성
  Flask-Migrate(Alembic) 기반 → 버전 업그레이드 시에도 사용

superset fab create-admin
  Flask-AppBuilder(fab) 로 관리자 계정 생성
  처음 한 번만 실행 (이미 있으면 에러)
  이후 추가 계정은 Superset UI → Settings → Users 에서 관리

superset init
  Admin / Alpha / Gamma / Public 기본 역할 생성
  각 역할에 기본 권한 할당
  반드시 create-admin 이후에 실행 (순서 바뀌면 권한 설정 꼬임)
```

---

---

# ⑤ 접속 확인

```
초기화 완료 후 브라우저에서 접속

pip 설치:  http://localhost:8088
Docker:    http://localhost:8088  (포트 매핑 8088:8088 기준)

ID: admin
PW: create-admin 에서 설정한 값
```

---

---

# 트러블슈팅

|증상|원인|해결|
|---|---|---|
|접속했는데 로그인 화면 안 나옴|`db upgrade` 누락|`superset db upgrade` 실행|
|로그인 후 권한 에러|`superset init` 누락|`superset init` 실행|
|`create-admin` 에러 — 이미 존재|이전에 이미 생성됨|UI → Settings → Users 에서 관리|
|Docker 재시작 후 계정 사라짐|`superset_data` 볼륨 없음|`docker-compose.yml` 에 `superset_data` 볼륨 추가|