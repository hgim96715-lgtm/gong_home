---
aliases:
  - sys모듈
  - 시스템제어
  - 표준출력
tags:
  - Python
related:
  - "[[Python_Logging]]"
  - "[[Python_Builtin_Functions]]"
  - "[[00_Python_HomePage]]"
  - "[[Python_Modules_Imports]]"
---
## Concept Summary

**"파이썬 인터프리터(실행기)를 직접 제어하는 도구."** 
파이썬 스크립트가 실행되는 **환경(System)** 정보를 얻거나, 스크립트의 실행을 강제로 멈추거나, 입력/출력 방향을 바꿀 때 사용합니다.

---
## `sys.stdout` (표준 출력) 

```python
#
logging.basicConfig(..., stream=sys.stdout)
```

- **의미:** "Standard Output(표준 출력)"의 약자입니다.
- **역할:** `print()` 함수가 글자를 내보내는 **기본 통로(모니터/터미널)** 입니다.
- **왜 굳이 이걸 썼나요?**
	- 도커(Docker) 환경에서는 로그를 **파일**에 숨기지 않고 **화면(stdout)**에 쏴줘야, `docker logs` 명령어로 밖에서 볼 수 있기 때문입니다.

---
## `sys.argv` (실행 인자)

데이터 엔지니어링 스크립트 짤 때 가장 많이 씁니다.

- **의미:** "Argument Vector". 터미널에서 스크립트를 실행할 때 뒤에 붙인 단어들을 **리스트**로 받아옵니다.
- **사용 예시:** "개발 서버(`dev`)용으로 돌릴지, 운영 서버(`prod`)용으로 돌릴지 터미널에서 정하고 싶을 때."

```bash
# 터미널에서 이렇게 실행했다면:
python my_job.py prod 2024-01-01
```

```python
import sys

# sys.argv는 리스트다! ['my_job.py', 'prod', '2024-01-01']
print(sys.argv[0]) # 파일명: my_job.py
print(sys.argv[1]) # 첫 번째 인자: prod
print(sys.argv[2]) # 두 번째 인자: 2024-01-01

if sys.argv[1] == "prod":
    print("운영 서버 접속!")
```

---
## `sys.exit()` (강제 종료)

스크립트를 **즉시 멈추고 싶을 때** 씁니다.

- **역할:** 프로그램 강제 종료. (마치 전원 버튼 꾹 누르는 것과 같음)
- **사용 예시:** 중요한 설정 파일이 없거나, DB 연결이 끊겼을 때 "더 이상 못 해!" 하고 죽을 때.

```python
import sys

try:
    # 뭔가 위험한 작업
    connect_database()
except Exception:
    print("치명적인 에러! 시스템 종료!")
    sys.exit(1) # 0은 정상 종료, 1 이상은 에러로 인한 종료를 뜻함
    
print("이 줄은 절대 실행되지 않음") # 위에서 죽었으니까
```

---
## `sys.path` (모듈 경로) 

가끔 `import my_module` 했는데 **"Module Not Found"** 에러 뜰 때 있죠? 그때 확인하는 변수입니다.

- **역할:** 파이썬이 `import` 할 때 파일을 뒤지는 **폴더 목록(리스트)** 입니다.
- **활용:** 내가 만든 파일이 파이썬 눈에 안 보일 때, `sys.path.append("/내/폴더/경로")`로 강제로 추가해 줄 수 있습니다.
