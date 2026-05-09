---
aliases:
  - 디렉토리 명령어
  - mkdir
  - ls -F
  - touch
  - tree
  - cd
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_File_Move_Copy]]"
---
# Linux_Directory_Commands — 디렉토리 관리

## 한 줄 요약

```
mkdir  → 디렉토리 생성
cd     → 디렉토리 이동
ls -F  → 파일/폴더 구분해서 목록 보기
tree   → 전체 구조 시각화
```

---

---

# ① mkdir — 디렉토리 생성 ⭐️

```bash
# 단일 디렉토리
mkdir src

# 여러 개 한 번에
mkdir src config docs
# src/ config/ docs/ 세 개 동시 생성

# 중간 경로 포함 (없으면 자동 생성)
mkdir -p project/src/utils
#  -p = parents (부모 디렉토리도 함께 생성)

# 권한 지정하며 생성
mkdir -m 755 mydir
```

## 프로젝트 구조 한 번에 생성

```bash
# 데이터 엔지니어 프로젝트 구조
mkdir -p myproject/{src,config,docs,logs,tests}

ls -F myproject/
# config/  docs/  logs/  src/  tests/
```

---

---

# ② touch — 빈 파일 생성 ⭐️

```bash
# 빈 파일 생성
touch hello.txt
touch file1.txt file2.txt file3.txt   # 여러 개 동시

# 파일 있으면 수정 시간만 업데이트
touch existing_file.txt

# 생성 확인
ls -l hello.txt
```

```
touch 두 가지 역할:
  파일 없음 → 빈 파일 새로 생성
  파일 있음 → 수정 시간 업데이트

빈 설정 파일 / 테스트 파일 만들 때 자주 씀
```

## 중괄호 확장 — 한 번에 여러 파일 ⭐️

```bash
# {1..5} = 1부터 5까지 숫자 자동 생성
touch note_{1..5}.txt
ls note*
# note_1.txt  note_2.txt  note_3.txt  note_4.txt  note_5.txt

# 응용 패턴
touch file_{1..10}.log          # file_1.log ~ file_10.log
mkdir dir_{a..e}                # dir_a ~ dir_e
touch report_{2020..2026}.txt   # 연도별 파일
```

```
{1..5}             숫자 범위 (1, 2, 3, 4, 5)
{a..e}             알파벳 범위 (a, b, c, d, e)
{src,config,docs}  직접 나열

touch note_{1..5}.txt 내부 동작:
  → touch note_1.txt note_2.txt note_3.txt note_4.txt note_5.txt
  쉘이 자동으로 펼쳐서 실행
```

---

---

# ③ ls — 목록 확인 ⭐️

```bash
ls              # 기본 목록
ls -l           # 상세 정보 (권한 / 크기 / 날짜)
ls -a           # 숨김 파일 포함 (. 으로 시작)
ls -la          # 상세 + 숨김 파일
ls -F           # 파일 종류 구분 기호 추가
ls -lh          # 파일 크기 사람이 읽기 좋게 (KB, MB)
ls -lt          # 수정 시간 순 정렬 (최신 먼저)
ls -lS          # 크기 순 정렬 (큰 것 먼저)
```

## ls -F 출력 해석 ⭐️

```bash
ls -F
# config/   docs/   logs/   src/   README.md   run.py*
#     ↑           ↑                    ↑(없음)     ↑
#  디렉토리    디렉토리              일반 파일    실행 파일

기호 의미:
  /   = 디렉토리
  *   = 실행 가능한 파일
  @   = 심볼릭 링크
  없음 = 일반 파일
```

---

---

# ③ cd — 디렉토리 이동

```bash
cd /home/labex/project    # 절대 경로로 이동
cd project                # 현재 위치에서 상대 경로
cd ..                     # 한 단계 위로
cd ../..                  # 두 단계 위로
cd ~                      # 홈 디렉토리로
cd -                      # 직전 디렉토리로 (토글)
cd                        # 홈 디렉토리로 (~ 생략)
```

---

---

# ④ pwd — 현재 위치 확인

```bash
pwd
# /home/labex/project/phoenix_project
```

---

---

# ⑤ tree — 전체 구조 시각화

```bash
# 설치 (없으면)
sudo apt-get install tree

# 기본
tree

# 특정 디렉토리
tree ~/project

# 깊이 제한
tree -L 2       # 2단계까지만

# 숨김 파일 포함
tree -a

# 디렉토리만
tree -d
```

```
tree 출력 예시:
phoenix_project/
├── config/
│   ├── config.json
│   └── config.json.bak
├── docs/
│   └── README.md
└── src/
    └── main_app.py
```

---

---

# ⑥ 실전 패턴 — 프로젝트 구조 설계

```bash
# 1. 프로젝트 루트 이동
cd ~/project/phoenix_project

# 2. 기본 폴더 구조 생성
mkdir src config docs

# 3. 생성 확인 (파일/폴더 구분)
ls -F
# config/  docs/  src/

# 4. 이동하면서 작업
cd config
pwd    # 현재 위치 확인

# 5. 전체 구조 확인
cd ~/project/phoenix_project
tree -L 2
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`mkdir a/b/c` 에러|중간 경로 없음|`mkdir -p a/b/c`|
|디렉토리인지 파일인지 헷갈림|ls 기본 출력|`ls -F` 로 `/` 확인|
|잘못된 위치에서 작업|pwd 안 함|작업 전 `pwd` 습관화|