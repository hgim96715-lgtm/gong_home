---
aliases:
  - 환경변수
  - Environment Variables
tags:
  - Linux
related:
  - "[[00_Linux_HomePage(기존)]]"
---
## 개념 한 줄 요약

환경 변수(Environment Variable)는 리눅스 시스템에서 프로세스가 실행되는 '주변 환경'을 정의하며, 프로그램들이 참고할 수 있도록 미리 저장해둔 **Key-Value(키-값) 형태의 설정 데이터**입니다.

---
## 왜 필요한가 (Why)

**문제 상황**:

- 실행 파일의 **절대 경로를 매번 입력**해야 한다 (예: `/Users/me/apps/python/bin/python`).
- **비밀번호·토큰을 코드에 하드코딩**하면 유출 위험이 크다.
- 환경(dev/stage/prod)마다 **설정이 달라 코드가 자주 바뀐다**.

**해결책**:
- **설정의 중앙화**: `PATH`에 경로를 등록하면 파일명만으로 실행 가능.
- **보안/유연성**: 코드는 변수명만 참조, 실제 값은 환경별로 주입.
- **배포 친화적**: 동일한 코드로 여러 환경 운영.

---
## 실무 적용 예시 (Practical Context)

### 데이터 엔지니어링에서 자주 쓰는 변수

- **시스템 구성**: `PATH`, `JAVA_HOME`, `SPARK_HOME`, `HADOOP_HOME`
- **애플리케이션 설정**: `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`
- **플랫폼 연동**: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `GCP_PROJECT`
- **사용자 환경**: `HOME`, `EDITOR`, `LANG`

---
## 코드 코어 포인트 (Core Points)

- **조회**: `env`(전체), `printenv VAR`, `echo $VAR`
- **설정**: `export VAR=value` → **자식 프로세스까지 전달**
- **영구 저장**: macOS(zsh)는 `~/.zshrc`, Linux(bash)는 `~/.bashrc`

---
## 상세 분석

### 환경 변수 확인

```bash
# 1. 모든 환경 변수와 그 값을 리스트로 출력합니다.
env

# 2. 특정 변수의 값만 찍어서 확인하고 싶을 땐 $ 기호를 붙입니다.
echo $HOME  # 결과: /Users/yourname

# 3. 쉘 스크립트 내부에서도 변수 이름 앞에 $를 붙여서 값을 가져옵니다.
if [ "$USER" == "root" ]; then
    echo "당신은 관리자입니다."
fi
```

### 환경 변수 생성 및 내보내기 (Export)

```bash
# 1. 일시적인 변수 만들기 (현재 쉘 세션에서만 유효, 자식 프로세스는 모름)
MY_VAR="Temporary Value"

# 2. 환경 변수로 내보내기 (이 터미널에서 실행할 모든 프로그램이 알 수 있게 함)
export MY_VARIABLE="Hello, World!"

# 3. 이미 존재하는 변수를 환경 변수로 승격시키기
MY_NAME="Gemini"
export MY_NAME
```

> **차이 핵심**: `export` 여부. 파이썬/자바/도커가 읽을 수 있느냐의 차이.

### 영구 저장 (Mac 기준)

- 터미널에서 친 `export` 명령어는 창을 닫으면 사라집니다. 
- 맥북(zsh 사용)에서는 아래 파일에 적어줘야 합니다.

```bash
# 1. 설정 파일을 엽니다 (Mac은 보통 .zshrc 를 사용합니다)
nano ~/.zshrc

# 2. 파일 맨 아래에 export 문을 추가하고 저장합니다.
# export DATABASE_URL="postgres://localhost:5432/airflow"

# 3. 변경 내용을 즉시 적용합니다.
source ~/.zshrc
```

---
## 초보자 들이 

1. **`$` 기호의 위치**
	- 변수를 정의할 때(왼쪽)는 `$`를 쓰지 않고, 그 값을 불러올 때(오른쪽)만 `$`를 씁니다.
    - ❌ `$MY_VAR="123"`
    - ✅ `MY_VAR="123"` -> `echo $MY_VAR`

2. **`export`를 안 하면 어떻게 되나요?**
	- 그냥 `MY_VAR="123"`이라고만 하면 내가 실행한 파이썬 스크립트는 그 변수의 존재를 모릅니다. 
	- 반드시 **`export`** 를 해줘야 '자식' 프로그램들이 물려받을 수 있습니다.
    
3.  **수정했는데 왜 안 바뀌죠?**: 
	- 설정 파일(`.zshrc` 등)을 고친 후에는 반드시 터미널을 새로 열거나 **`source`** 명령어로 새로고침을 해줘야 합니다.