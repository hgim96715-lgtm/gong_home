---
aliases:
  - Shell Input
  - read command
  - stdin
  - positional parameters
  - ㅅ
tags:
  - Linux
related:
  - "[[Shell_Scripting_Basics]]"
  - "[[Redirection_Pipe]]"
---
## 개념 한 줄 요약 ⚡️

**"스크립트가 실행 도중에 멈춰서 사용자의 입력을 기다리게 하거나(`read`), 실행할 때 미리 던져준 재료(`Arguments`)를 받는 방법."**

---
##  멈춰서 듣기: `read` (Interactive) 

스크립트 실행 중에 커서가 깜빡거리며 **"타이핑할 때까지 기다리는"** 상태가 됩니다.

### ① 기본 문법

```bash
read 변수명
```

- 사용자가 입력한 값이 `변수명`에 저장됩니다.

### ② 실전 문제 해결 (User's Code) 

문제: "입력을 받아서 `Welcome (이름)`을 출력하라."

```bash
# 1. 사용자의 입력을 기다림 -> 입력값을 'name' 변수에 저장
read name

# 2. 저장된 변수를 출력
echo "Welcome $name"
```

### ③ 꿀팁 옵션 (`-p`, `-s`)

그냥 커서만 깜빡이면 뭐 하라는 건지 모르니까 안내 문구를 띄울 수 있습니다.

```bash
# -p (prompt): 안내 문구 띄우기
read -p "이름을 입력하세요: " name

# -s (silent): 비밀번호 입력 (화면에 글자가 안 보임) 🔒
read -s -p "비밀번호: " password
```

---
## 던져서 받기: Positional Parameters (`$1`) 

`read`는 스크립트 도중에 멈추지만, **자동화(Automation)** 를 할 때는 멈추면 안 되는 경우가 많습니다. 
그래서 실행할 때 아예 값을 던져줍니다.

### ① 개념

- **`$1`**: 첫 번째 재료 (Argument 1)
- **`$2`**: 두 번째 재료 (Argument 2)
- **`$0`**: 파일 이름 그 자체

### ② 예제

```python
# script.sh 파일 내용
echo "Welcome $1"
```

실행 방법:

```bash
./script.sh "Alice"
# 결과: Welcome Alice
```

**차이점:** `read`는 스크립트 안에서 `name=` 하고 기다리는 것이고, `$1`은 밖에서 `Alice`를 던져주는 것입니다.

---
## 표준 입력 (stdin) 파이프 연결 

`read`는 키보드 입력뿐만 아니라, **파이프(`|`)로 넘어온 데이터**도 읽을 수 있습니다.

```bash
# "Alice"라는 글자를 파이프로 넘겨서 read가 읽게 함
echo "Alice" | (read name; echo "Welcome $name")
# 결과: Welcome Alice
```

**코딩 테스트 팁:** 
> HackerRank 같은 사이트는 보통 파이프(`|`)나 리다이렉션(`<`)을 통해 데이터를 스크립트에 밀어 넣습니다. 
> 이때 `read` 명령어는 밀려들어오는 데이터를 **한 줄씩** 읽어냅니다.