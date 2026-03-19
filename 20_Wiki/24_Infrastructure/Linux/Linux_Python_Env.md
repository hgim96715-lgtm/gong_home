---
aliases:
  - pip
  - venu
  - pyenv
  - requiremnets.txt
  - 파이썬 환경
  - 가상 환경
tags:
  - Linux
  - Python
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Package_APT]]"
  - "[[Docker_Python_Requirements]]"
  - "[[Python_Virtual_Env]]"
---

# Linux_Python_Env — 파이썬 환경 관리

## 한 줄 요약

```
pip       패키지 설치 도구
venv      프로젝트별 격리된 파이썬 환경
pyenv     파이썬 버전 관리
requirements.txt  패키지 목록 파일
```

---

---

# ① pip — 패키지 설치

```bash
# 패키지 설치
pip install requests
pip install kafka-python==2.0.2   # 버전 고정
pip install requests kafka-python python-dotenv  # 여러 개 한번에

# 패키지 삭제
pip uninstall requests

# 설치된 패키지 목록
pip list
pip list | grep kafka

# 패키지 정보 확인
pip show requests

# 업그레이드
pip install --upgrade requests
```

## pip vs pip3

```
파이썬 2 가 설치된 환경:
  pip   → Python 2 용
  pip3  → Python 3 용

파이썬 3 만 있는 현대 환경:
  pip = pip3 (동일)
  → 명확히 하려면 pip3 사용 권장

Docker 컨테이너 (Ubuntu):
  pip3 install ... 로 써야 안전
```

## pip 설치 시스템 패키지 충돌 (Linux)

```bash
# Ubuntu 에서 pip install 시 에러:
# error: externally-managed-environment

# 해결 1: --break-system-packages 플래그
pip3 install kafka-python --break-system-packages

# 해결 2: 가상환경 사용 (권장)
python3 -m venv .venv
source .venv/bin/activate
pip install kafka-python
```

---

---

# ② requirements.txt — 패키지 목록 파일

```
프로젝트에서 필요한 패키지와 버전을 파일로 관리
협업 / 배포 / Docker 빌드 시 동일한 환경 재현
```

## 작성법

```
# requirements.txt
requests==2.31.0
kafka-python==2.0.2
python-dotenv==1.0.0
pandas>=2.0.0          # 최소 버전 이상
numpy                  # 버전 고정 없음 (최신)
```

```bash
# requirements.txt 로 한번에 설치
pip install -r requirements.txt
pip3 install -r requirements.txt --break-system-packages
```

## 현재 환경에서 추출

```bash
# 설치된 패키지 전체를 requirements.txt 로 저장
pip freeze > requirements.txt

# 가상환경 활성화 후 freeze 하면 해당 환경만 추출
source .venv/bin/activate
pip freeze > requirements.txt
```

```
pip freeze 주의:
  전역 환경에서 실행하면 불필요한 패키지까지 전부 포함
  → 가상환경(venv) 안에서 freeze 하는 것이 좋음
```

---

---

# ③ venv — 프로젝트별 가상환경

```
프로젝트마다 독립된 파이썬 환경 생성
A 프로젝트: requests 2.28 / B 프로젝트: requests 2.31 → 충돌 없음

왜 필요한가:
  전역 pip install 하면 시스템 전체에 설치됨
  프로젝트별 버전이 다르면 충돌
  → venv 로 프로젝트마다 격리
```

## 기본 사용법

```bash
# 가상환경 생성
python3 -m venv .venv          # .venv 폴더에 생성
python3 -m venv myenv          # myenv 폴더에 생성

# 가상환경 활성화
source .venv/bin/activate      # Linux / macOS
.venv\Scripts\activate         # Windows

# 활성화 확인 → 프롬프트 앞에 (.venv) 표시
(.venv) user@host:~$

# 패키지 설치 (가상환경 안에만 설치됨)
pip install requests kafka-python

# 가상환경 비활성화
deactivate
```

## 프로젝트 구조 (예시)

```
hospital-project/
├── .venv/              ← 가상환경 (gitignore 에 추가)
├── producer.py
├── requirements.txt
└── .gitignore          ← .venv/ 추가
```

```gitignore
# .gitignore
.venv/
__pycache__/
*.pyc
.env
```

---

---

# ④ pyenv — 파이썬 버전 관리

```
시스템에 파이썬 여러 버전 설치 + 프로젝트별 버전 지정

venv  패키지 격리 (같은 파이썬 버전)
pyenv 파이썬 버전 자체를 바꿈

A 프로젝트: Python 3.10
B 프로젝트: Python 3.12
→ pyenv 로 프로젝트마다 버전 다르게
```

## 버전 선택 기준

```
3.9   구버전 (ET.Element | None 같은 문법 불가)
3.10  | 연산자 사용 가능한 최소 버전
3.11  ← 권장  안정적 + 속도 개선 + 라이브러리 호환성 좋음
3.12  최신  일부 라이브러리 아직 미지원 가능성
```

## 설치

```bash
# macOS
brew install pyenv

# .zshrc 에 추가 (없으면 pyenv 명령어 인식 안 됨)
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.zshrc
echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.zshrc
echo 'eval "$(pyenv init -)"' >> ~/.zshrc
source ~/.zshrc   # 적용

# Linux
curl https://pyenv.run | bash
# .bashrc 에 동일하게 추가
```

## 버전 확인 & 변경 — macOS 실전

```bash
# 현재 버전 확인
python3 --version

# 설치 가능한 버전 목록
pyenv install --list | grep "3.11"

# 3.11.9 설치
pyenv install 3.11.9

# 설치된 버전 목록
pyenv versions
# * system  ← 현재 사용 중
#   3.11.9

# 전역 버전 변경
pyenv global 3.11.9

# 확인
python3 --version   # Python 3.11.9
```

```
⚠️ pyenv global 변경 후에도 python3 --version 이 안 바뀌면:
  터미널 재시작 또는 source ~/.zshrc 실행
  which python3 로 경로 확인
  /Users/이름/.pyenv/shims/python3 이면 정상
```

## 프로젝트별 버전 지정


```bash
# 현재 폴더에서만 버전 지정 (.python-version 파일 생성)
cd hospital-project
pyenv local 3.11.9

# 확인
cat .python-version   # 3.11.9
python3 --version     # Python 3.11.9
```

## pyenv + venv 조합


```bash
pyenv local 3.11.9
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

## 되돌리기 & 우선순위

```bash
# 시스템 Python 으로 되돌리기
pyenv global system
# → pyenv 아닌 원래 macOS 기본 Python 사용

# 특정 폴더의 local 버전 제거
rm .python-version
```

```
pyenv 우선순위:
  1. pyenv shell   현재 세션에서만 (pyenv shell 3.11.9)
  2. pyenv local   해당 폴더 (.python-version 파일)
  3. pyenv global  전체 기본값

global 지워도 폴더에 .python-version 있으면 그게 우선 적용
언제든지 pyenv global system 으로 원래대로 복구 가능
```


---

---

# ⑤ 상황별 선택

|상황|도구|
|---|---|
|패키지 설치|`pip install`|
|패키지 목록 공유|`requirements.txt`|
|프로젝트별 패키지 격리|`venv`|
|파이썬 버전 다르게 써야 할 때|`pyenv`|
|Docker 환경|`pip install -r requirements.txt`|

```
DE 실무 흐름:
  pyenv 로 Python 버전 고정
  → venv 로 프로젝트 격리
  → pip install 로 패키지 설치
  → pip freeze > requirements.txt 로 목록 저장
  → Docker / 배포 시 pip install -r requirements.txt
```

---

---

# 트러블슈팅

|증상|원인|해결|
|---|---|---|
|`externally-managed-environment`|Ubuntu pip 보호 정책|`--break-system-packages` 또는 venv 사용|
|`pip: command not found`|pip 미설치|`sudo apt install python3-pip`|
|`ModuleNotFoundError`|가상환경 비활성화 상태|`source .venv/bin/activate`|
|Docker 에서 패키지 없음|requirements.txt 누락|Dockerfile / command 에 `pip install -r requirements.txt`|
|`pip freeze` 에 너무 많은 패키지|전역 환경에서 실행|venv 활성화 후 `pip freeze`|