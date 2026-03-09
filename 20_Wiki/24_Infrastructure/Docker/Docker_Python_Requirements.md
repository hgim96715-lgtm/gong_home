---
aliases:
  - requirements
  - pip의존성
  - 파이썬 패키지관리
tags:
  - Docker
related:
  - "[[00_Docker_HomePage]]"
  - "[[Dockerfile_Basics]]"
  - "[[Docker_Image_Layers]]"
---

# 🐳 Docker_Python_Requirements

## 개념 한 줄 요약

> **"이 프로젝트를 실행하려면 어떤 라이브러리가 필요한지 적어둔 명세서."** 
> Dockerfile 에서 `COPY requirements.txt` 하고 `RUN pip install` 로 설치한다.

```
requirements.txt  →  Dockerfile 이 읽어서  →  pip install  →  이미지 안에 설치됨
```

---

---

# ① requirements.txt 만드는 법

## 직접 작성 (버전 고정) ⭐️ 권장

```
kafka-python==2.0.2
psycopg2-binary==2.9.9
requests==2.31.0
python-dotenv==1.0.0
```

## pip freeze 로 자동 생성

```bash
# 현재 환경에 설치된 패키지 전부 추출
$ pip freeze > requirements.txt
```

```
⚠️  pip freeze 주의사항:
현재 환경의 모든 패키지가 다 들어감 (불필요한 것까지)
버전이 너무 엄격하게 고정되어 충돌 날 수 있음
→ 직접 필요한 것만 적는 게 깔끔함
```

---

---

# ② 버전 표기 방식

```
kafka-python==2.0.2   →  정확히 이 버전만 (가장 엄격)
kafka-python>=2.0.2   →  이 버전 이상
kafka-python>=2.0.2,<3.0.0  →  2.x 버전만 허용
kafka-python          →  버전 상관없이 최신 (비권장 ⚠️)
```

```
Docker 이미지는 버전 고정 == 을 권장
버전 없으면 빌드할 때마다 다른 버전이 설치될 수 있음
→ "어제는 됐는데 오늘은 안 돼" 의 원인
```

---

---

# ③ Dockerfile 에서 사용하는 법

```dockerfile
WORKDIR /app

# ✅ requirements.txt 먼저 복사 + 설치 (캐시 최적화)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 그 다음 소스코드 복사
COPY . .
```

```
--no-cache-dir  →  pip 캐시 저장 안 함 → 이미지 용량 절약
-r              →  requirements.txt 파일을 읽어서 설치
```

---

---

# ④ 패키지 설치 & 관리 명령어

```bash
# 설치
$ pip install kafka-python

# 특정 버전 설치
$ pip install kafka-python==2.0.2

# requirements.txt 한 번에 설치
$ pip install -r requirements.txt

# 설치된 패키지 목록 확인
$ pip list

# 특정 패키지 정보 확인
$ pip show kafka-python

# 업그레이드
$ pip install --upgrade kafka-python

# 삭제
$ pip uninstall kafka-python
```

---

---

# 초보자 실수 체크리스트

| 실수                            | 원인                         | 해결                                                        |
| ----------------------------- | -------------------------- | --------------------------------------------------------- |
| 빌드할 때마다 버전이 달라짐               | 버전 고정 안 함                  | `{text}==` 으로 버전 명시                                       |
| 코드 한 줄 바꿨는데 pip install 다시 실행 | COPY 순서 잘못됨                | `requirements.txt` 먼저 COPY 후 pip install, 그 다음 `COPY . .` |
| 이미지 용량이 너무 큼                  | pip 캐시가 이미지에 포함됨           | `--no-cache-dir` 옵션 추가                                    |
| 로컬엔 되는데 컨테이너에서 안 됨            | requirements.txt 에 누락된 패키지 | 로컬에서 `pip freeze` 로 확인 후 추가                               |

---
