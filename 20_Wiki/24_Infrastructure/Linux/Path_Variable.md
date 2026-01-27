---
aliases:
  - PATH 변수
  - Command Not Found 해결
  - 경로 설정
  - 경로 우선순위
tags:
  - Linux
  - PATH
related:
  - "[[Environment_Variables]]"
  - "[[Shell_Config]]"
  - "[[00_Linux_HomePage]]"
---
## 1. 개념 한 줄 요약

**"시스템이 터미널에 입력된 명령어(실행 파일)를 찾으러 돌아다니는 '탐색 지도(Map)' 목록이다."**
내가 `python`이라고 외치면, 컴퓨터는 이 목록에 적힌 폴더들을 **순서대로** 뒤져서 실행 파일을 찾는다.

---
## 왜 필요한가? (Why)

**문제점:**
- 원래 프로그램을 실행하려면 `/usr/local/bin/python3.9` 처럼 **절대 경로(Full Path)** 를 입력해야 한다.
- 매번 이렇게 치는 건 인간이 할 짓이 아니다.

**해결책:**
- "야, 내가 그냥 `python`이라고만 해도, 네가 알아서 `PATH`에 적힌 폴더들 뒤져서 찾아와!" 라고 약속해두는 것이다.
- 덕분에 우리는 긴 경로를 외울 필요 없이 프로그램 이름만 쳐도 실행할 수 있다.

---
## 3. 실무 적용 사례 (Practical Context)

데이터 엔지니어링에서 가장 흔한 에러 1위: **`zsh: command not found: java`**

1.  **원인:** Java를 설치하긴 했는데, 그 설치된 폴더 위치를 `PATH` 변수에 추가해주지 않아서 컴퓨터가 못 찾는 것이다.
2.  **해결:** 설치 경로(예: `/opt/java/bin`)를 `PATH`에 슬쩍 끼워 넣어주면 해결된다.

---
## Code Core Points (핵심 문법)

1.  **구분자:** 여러 폴더는 **콜론(`:`)** 으로 구분한다. (띄어쓰기 아님!)
2.  **우선순위:** **왼쪽(앞)** 에 있는 경로가 대장이다. (같은 이름의 파일이 있으면 먼저 발견된 놈이 실행됨)
3.  **보존(Append):** 기존의 `PATH`를 지우지 않으려면 반드시 **`$PATH`를 포함**해서 덧붙여야 한다.

---
## 상세 코드 분석 (Detailed Analysis)

### ① 현재 경로 확인하기

```bash
echo $PATH
# 출력: /usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin
# 해석: "명령어를 치면 1번방(/usr/local/bin) 찾아보고, 없으면 2번방(/usr/bin) 가보고..."
```

### ② 경로 추가하기 (표준 방식) ⭐️ 가장 중요!

"기존 목록($PATH) 뒤에 내꺼(:/my/tools)를 붙여라"

```bash
# [안전한 방법] 기존 PATH를 살리면서 뒤에 추가
export PATH="$PATH:/Users/gong/my_tools/bin"
```

### ③ 우선순위 바꾸기 (새치기)

**"내꺼(/my/new/python)를 제일 먼저 찾아봐!"** (버전 관리할 때 씀)

```bash
# 내 경로를 $PATH 보다 앞에 둠
export PATH="/my/new/python/bin:$PATH"
```

### ④ 검증하기 (`which`)

도대체 어떤 놈이 실행되고 있는지 궁금할 때 쓴다.

```bash
which python
# 출력: /usr/bin/python (아, 시스템 기본 파이썬이구나)
# 출력: /my/new/python/bin/python (오, 내가 깐 게 실행되네?)
```

---
## 초보자가 자주 하는 실수 (Misconceptions)

### ① "기존 경로를 날려먹었어요!"  (최악의 실수)

```bash
# [절대 금지] $PATH를 빼먹으면 기존 경로가 다 사라짐
export PATH="/new/path"
```

- **증상:** `ls`, `vi`, `cat` 명령어가 다 안 먹힘. (`command not found`)
- **이유:** `ls` 명령어도 `/bin`이라는 폴더에 있는데, `PATH`에서 `/bin`을 지워버렸으니 못 찾는 것.
- **응급처치:** 터미널을 끄지 말고, 임시로 절대 경로를 쳐서 복구해야 함.
    - `/bin/vi ~/.zshrc` (절대 경로로 vi 실행해서 설정 파일 수정)

### ② "오타 주의"

- 윈도우 습관 때문에 구분자로 세미콜론(`;`)을 쓰면 안 된다. 리눅스는 **콜론(`:`)** 이다.
- 공백이 들어가면 안 된다. `PATH = ...` (X) -> `PATH=...` (O)

---
## 내가 겪은 에러 

**상황:** `.zshrc`에서 PATH 설정을 잘못 건드린 후, `source`를 했더니 갑자기 터미널이 먹통이 됨.

**1. `zsh: command not found: clear`**
* **원인:** `clear` 명령어는 원래 `/usr/bin/clear`에 위치한다. 하지만 `PATH` 변수에서 `/usr/bin` 경로가 사라져서(덮어씌워져서) 시스템이 `clear`를 못 찾게 된 것이다.

**2. `prompt_status:9: command not found: wc`**
* **원인:** 터미널의 프롬프트(입력창 모양)를 꾸며주는 테마 프로그램이 내부적으로 글자 수를 세기 위해 **`wc` (Word Count)** 명령어를 사용한다.
* PATH가 끊기니 이 내부 프로그램조차 `wc`를 못 찾아서 연쇄적으로 에러가 터진 것이다.

**-> 교훈:** PATH를 설정할 땐 반드시 **`$PATH:`** 를 포함해서 기존 경로를 살려야 한다!


```bash
# 1단계: (기본 경로를 강제로 주입하는 겁니다.)
export PATH=/usr/bin:/bin:/usr/sbin:/sbin
# 2단계 다시 설정 파일을 엽니다.
vi ~/.zshrc

# 3단계 코드 수정
# 기존의 `$PATH`를 안 적고 덮어써서 사고가 난 겁니다.)
# export PATH="/어쩌구/저쩌구" <-- 이렇게 샵(#)을 붙이세요!

# 4단계 적용
source ~/.zshrc
```

