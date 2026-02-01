---
aliases:
  - 쉘스크립트
  - bash
  - shell_script
  - if문
  - for문
  - while문
  - 쉘변수
tags:
  - Shell
  - Linux
related:
  - "[[Linux_Scheduling_Crontab]]"
  - "[[File_Management]]"
  - "[[00_Linux_HomePage]]"
---
$##  개념 한 줄 요약

**"리눅스 명령어들을 파일 하나에 모아서 순서대로 실행시키는 '자동화 대본(Script)'."**

* **Bash (Bourne Again SHell):** 리눅스와 macOS의 기본 쉘이자, 가장 강력한 명령어 해석기입니다
* **쉘 스크립트:** `.sh` 확장자를 가진 파일에 명령어와 로직(if, for)을 작성하여 복잡한 작업을 한 방에 처리합니다.

>Bash에서는 명령어와 옵션 사이 공백이 생명

---
##  왜 필요한가? (Why)

**문제점:**
- "매일 아침 서버 상태를 체크하려고 명령어 10개를 매번 타이핑하고 있어요."
- "파일 100개의 이름을 일괄 변경해야 하는데, 하나씩 `mv` 하려니 손가락이 아파요."

**해결책:**
- **쉘 스크립트**를 작성해두면 `./myscript.sh` 엔터 한 번으로 100가지 일을 순서대로 처리합니다.
- 변수와 조건문을 사용해 상황에 따라 똑똑하게 동작하는 자동화 도구를 만들 수 있습니다.

---
## Code Core Points: ① 기본 구조와 실행

### 스크립트의 시작 (Shebang)

모든 쉘 스크립트의 첫 줄은 반드시 이렇게 시작해야 합니다.

```bash
#!/bin/bash
# 위 코드는 "이 파일은 /bin/bash를 사용해 실행하라"는 뜻입니다

# 주석(Comment)은 #으로 시작합니다
echo "Hello, World!"
```

### 실행 방법 (Permission) 

파일을 그냥 만들면 실행이 안 됩니다. **실행 권한(`x`)** 을 줘야 합니다.

```bash
# 1. 실행 권한 부여
chmod +x my_script.sh 

# 2. 스크립트 실행 (현재 경로 ./ 필수)
./my_script.sh 
```

---
## Code Core Points: ② 변수 (Variables)

데이터를 담는 그릇입니다. 
**만들 때는 그냥 쓰고, 쓸 때는 `$`를 붙입니다.**

```bash
#!/bin/bash

# 1. 변수 선언 (주의: 등호 = 앞뒤에 공백 절대 금지!) 🚫
name="Alice" 

# 2. 변수 사용 ($ 기호 사용)
echo "Hello, $name!" 
```

---
## Code Core Points: ③ 조건문 (If / Else)

"오전이면 굿모닝, 오후면 굿이브닝" 처럼 상황을 판단합니다. 
**대괄호 `[ ]` 안쪽 양옆에 반드시 공백이 있어야 합니다.**

[문법 공식] ` if [ 조건 ]; then ... elif ... else ... fi`

```bash
#!/bin/bash

hour=$(date +%H) # 현재 시간을 변수에 저장 [cite: 435]

if [ "$hour" -lt 12 ]; then        # 12보다 작으면 (Less Than)
    echo "Good morning!"
elif [ "$hour" -lt 18 ]; then      # 18보다 작으면
    echo "Good afternoon!"
else                               # 그 외
    echo "Good evening!"
fi
```

### 비교 연산자 치트시트

- **숫자 비교:** `-eq`(같음), `-ne`(다름), `-gt`(초과), `-lt`(미만), `-ge`(이상), `-le`(이하)
- **문자 비교:** `{text}=` (같음), `!=` (다름), `-z` (비어있음)
- **파일 검사:** `-f` (파일있음?), `-d` (디렉토리있음?)

>**`fi`**: `if`를 거꾸로 쓴 것입니다. **"여기서 if 문 끝!"** 이라고 닫아주는 괄호 `}` 역할입니다.
>**`if ...; then`**: "만약 조건이 맞다면(if), **그럼(then)** 이걸 실행해."라는 뜻입니다. (같은 줄에 쓸 땐 `;` 필수!)

---
## Code Core Points: ④ 반복문 (Loops)

똑같은 일을 여러 번 시킬 때 사용합니다.

반복할 구간을 **`do`** 와 **`done`** 으로 감싸줍니다.

### For 문 (리스트 반복)

```bash
# 1부터 5까지 숫자를 하나씩 꺼내서 i에 넣음
for i in 1 2 3 4 5; do # 반복 시작! (do)
    echo "Number: $i"
done 
```

### While 문 (조건 반복)

```bash
counter=0
# counter가 5보다 작은 동안 계속 반복
while [ "$counter" -lt 5 ]; do     # 조건이 참이면 해라(do)
    echo "Counter: $counter"
    counter=$((counter + 1)) # 1씩 증가
done [cite: 497-501]
```


>**`do`**: "자, 이제 반복 작업을 **시작(do)** 한다!"
> **`done`**: "반복 작업 **완료(done)**! 다시 위로 올라가거나 끝내."

---
## 실전 팁: 에러 처리 (Error Handling)

스크립트가 돌다가 중요 파일이 없으면 멈춰야겠죠?

```bash
file="important.txt"

# ! -f : 파일이 존재하지(!) 않으면 
if [ ! -f "$file" ]; then
    echo "Error: $file does not exist!"
    exit 1  # 1번 코드로 비정상 종료 알림 [cite: 540]
fi
```

---
## 초보자가 자주 하는 실수 (Misconceptions)

### ① "변수 선언할 때 띄어쓰기 했더니 에러나요."

- **Bash는 띄어쓰기에 매우 민감합니다.**
- `name = "Alice"` (X) -> 프로그램 `name`을 실행하라는 뜻으로 오해함.
- `name="Alice"` (O) -> 변수에 값을 넣음.


### ② "if 문에서 `[`랑 `]` 사이를 붙여 썼어요."

- `if ["$a" -eq 1];` (X) -> 에러 발생.
- `if [ "$a" -eq 1 ];` (O) -> 대괄호 안쪽은 **반드시 한 칸 띄어야 합니다.**

### ③ "스크립트 안에서 `source`는 뭔가요?"

- `./script.sh`는 새 창(Subshell)을 열어서 실행하고 꺼지지만,
- `source script.sh`는 **현재 창(Current Shell)** 에 설정을 바로 적용합니다.
- 환경 변수나 Alias 설정할 때 주로 씁니다.

