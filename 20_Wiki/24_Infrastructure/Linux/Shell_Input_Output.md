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
  - "[[00_Linux_HomePage]]"
---
## 개념 한 줄 요약 ⚡️

**"스크립트에게 말을 거는 방법(Input)과 스크립트가 대답하는 방법(Output)."**

* **Input:** 실행 도중에 물어보거나(`read`), 실행할 때 미리 던져주거나(`Arguments`).
* **Output:** 화면에 결과를 보여주는데, 변수 값을 쏙쏙 박아서 보여주기(`echo $var`).

---
##  멈춰서 듣기: `read` (Interactive) 

스크립트 실행 중에 커서가 깜빡거리며 **"사용자가 타이핑할 때까지 기다리는"** 상태가 됩니다.

### ① 기본 문법

```bash
read 변수명
```

- 사용자가 엔터를 칠 때까지 기다리고, 입력한 값을 `변수명`에 저장합니다.

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
# -p (prompt): 안내 문구 띄우기 (줄바꿈 없이 바로 옆에 입력 가능
read -p "이름을 입력하세요: " name

# -s (silent): 비밀번호 입력 (화면에 글자가 안 보임) 🔒
read -s -p "비밀번호: " password
```

### ④  핵심 원리: "한 줄 소비" (Stream Consumption)

`read`는 단순히 입력을 기다리는 게 아니라, **"표준 입력(Stdin) 스트림에서 한 줄을 뚝 떼어가는(Consume)"** 명령어입니다.

#### ❌ 흔한 실수 (The Trap)

파일 내용을 한 줄씩 읽는 `while` 반복문 안에서, 사용자에게 질문을 하려고 또 `read`를 쓰면?

```bash
# data.txt 파일 내용:
# 1. 사과
# 2. 바나나
# 3. 포도

cat data.txt | while read line
do
    echo "현재 과일: $line"
    
    # [문제 발생] 사용자의 입력을 기다리는 게 아니라,
    # 파이프로 들어오고 있는 data.txt의 '다음 줄(바나나)'을 훔쳐 먹어버립니다!
    read -p "계속 할까요? (y/n) " answer 
done
```

**결과:**

1. `while read line`이 "1. 사과"를 읽음.
2. 안쪽의 `read answer`가 "2. 바나나"를 **훔쳐서 `answer` 변수에 넣고 소비해버림.** (사용자 입력은 무시됨)
3. 다시 `while`로 돌아가니 "3. 포도"를 읽음.
4. **결국 "2. 바나나"는 출력되지 않고 증발함.**

#### ✅ 해결책 (Redirect)

사용자 입력(키보드)을 받고 싶다면, 표준 입력(`0`)을 강제로 **터미널(`/dev/tty`)** 로 지정해야 합니다.

```bash
# "야, 파일 내용 말고(Stdin), 키보드(TTY)에서 읽어와!"
read -p "계속 할까요? " answer < /dev/tty
```


---
## 던져서 받기: Positional Parameters (`$1`) 

`read`는 스크립트 도중에 멈추지만, **자동화(Automation)** 를 할 때는 멈추면 안 되는 경우가 많습니다. 
그래서 실행할 때 아예 값을 던져줍니다.

### ① 개념

- **`$0`**: 파일 이름 그 자체 (예: `script.sh`)
- **`$1`**: 첫 번째 재료 (Argument 1)
- **`$2`**: 두 번째 재료 (Argument 2)

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

**차이점:** 
- `read`: 식당 종업원이 와서 "주문하시겠어요?" 하고 기다리는 것.
- `$1`: 배달 앱으로 주문할 때 메뉴를 미리 찍어서 보내는 것 (기다림 없음, 빠름).
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

---
## 출력의 마법: `echo`와 변수 

`echo`는 단순히 고정된 글자만 찍는 게 아닙니다. `$` 기호를 붙이면 **변수 안에 저장된 값**을 꺼내서 보여줍니다.

### ① 문자열 vs 변수 (Variable Expansion)

- **그냥 쓰면:** 글자 그대로 출력됨.
- **`$`를 붙이면:** 변수 저장소를 뒤져서 **값**을 가져옴.

```bash
name="Gong"
sum=30

echo name    # 결과: name (그냥 글자)
echo $name   # 결과: Gong (변수 안의 값)
```

### ② 문장 속에 변수 섞어 쓰기

가장 많이 쓰는 방식입니다. **큰따옴표(`" "`)** 안에서는 `$`가 작동합니다.

```bash
echo "합계는 $sum 입니다."
# 결과: 합계는 30 입니다.
```

**주의: 따옴표의 종류가 중요합니다!**

- **큰따옴표 (`" "`)**: `$`를 변수로 해석해줍니다. (친절함) -> `값: 30`
- **작은따옴표 (`' '`)**: 모든 기호를 무시하고 문자 그대로 출력합니다. (엄격함) -> `값: $sum`

