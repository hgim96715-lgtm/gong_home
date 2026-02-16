---
aliases:
  - os 모듈
  - 환경변수
  - 파일삭제
  - 디렉토리 관리
  - os.path
tags:
  - Python
related:
  - "[[Python_Pathlib]]"
  - "[[Python_File_IO]]"
  - "[[Python_Sys_Module]]"
  - "[[00_Python_HomePage]]"
---
## 개념 한 줄 요약

**"파이썬이 내 컴퓨터(운영체제)의 파일, 폴더, 그리고 환경 변수까지 직접 주무를 수 있게 해주는 만능 도구"**

---
## 파일과 디렉토리 관리 (Pathlib의 조상)

`pathlib`가 나오기 전까지는 `os.path`가 파일 관리의 표준이었습니다. 레거시 코드에는 여전히 많이 쓰이므로 반드시 읽을 줄 알아야 합니다.

### ① 파일이 있나? 없나? (`exists`)

```python
import os

file_path = "data/user_log.csv"

# 파일(또는 폴더)이 실제로 존재하는지 확인 (True/False)
if os.path.exists(file_path):
    print("파일 찾았다!")
else:
    print("파일이 없습니다.")
```

### ② 파일 삭제 및 폴더 생성 (`remove`, `makedirs`)

```python
# 1. 파일 삭제 (복구 불가!)
os.remove("useless_file.txt")

# 2. 빈 폴더 삭제
os.rmdir("empty_folder")

# 3. 폴더 만들기 (하위 폴더까지 한 번에)
# exist_ok=True: 이미 있어도 에러 안 내고 넘어감 (꿀팁!)
os.makedirs("data/2026/02", exist_ok=True)
```

---
## 데이터 엔지니어의 필수템: 환경 변수 (`environ`) ⭐️

이 기능 때문에 `pathlib`가 있어도 `os` 모듈을 버릴 수 없습니다. 
**DB 비밀번호나 API 키** 같은 민감한 정보는 코드에 직접 적지 않고 **환경 변수**로 숨겨두고 가져옵니다.

```python
import os

# 1. 내 컴퓨터의 환경 변수 가져오기
# .get()을 쓰면 환경변수가 없을 때 에러 대신 None을 줍니다. (안전함)
db_password = os.environ.get("DB_PASSWORD")

# 2. 코드 안에서 임시로 환경 변수 설정하기
os.environ["KAFKA_TOPIC"] = "user-log"

print(f"현재 토픽: {os.environ['KAFKA_TOPIC']}")
```

---
## 경로 다루기 (`path`)

`pathlib`의 `/` 연산자가 나오기 전에는 `os.path.join`으로 경로를 붙였습니다.

```python
# [Old Style] 운영체제에 맞게 경로 합치기
# Windows는 역슬래시(\), Mac/Linux는 슬래시(/)를 알아서 써줌
full_path = os.path.join("data", "uploads", "image.png")

# 파일명과 확장자 분리
filename = os.path.basename(full_path) # image.png
folder = os.path.dirname(full_path)    # data/uploads
```

---
- **Pathlib vs OS**: 파일 경로는 `pathlib`가 훨씬 편하지만, **환경 변수(`os.environ`)** 와 **복잡한 시스템 제어**는 여전히 `os` 모듈의 영역입니다.
- **안전한 삭제**: `os.remove()`는 휴지통으로 가지 않고 바로 삭제되니 주의하세요!