---
aliases:
  - nohup
  - job_control
  - fg
  - bg
  - jobs
tags:
  - Linux
related:
  - "[[Process_Management]]"
  - "[[Process_vs_Thread]]"
  - "[[00_Linux_HomePage]]"
---
## 개념 한 줄 요약

**"내 눈앞(Foreground)에서 치워버리고, 뒤(Background)에서 조용히 일하게 만드는 기술."**

* **Foreground (전경):** 내가 명령어를 치고 끝날 때까지 멍하니 기다려야 하는 상태. (Ctrl+C 누르면 죽음)
* **Background (후경):** 명령어를 실행시켜 놓고 나는 다른 일을 할 수 있는 상태.
* **Nohup (No Hang Up):** 터미널을 꺼도 프로세스가 죽지 않게 "귀를 막아주는" 명령어.

---

## 왜 필요한가? (Why)

**문제점:**
- "데이터 100GB를 옮겨야 하는데 3시간 걸린대요. 터미널 켜두고 퇴근해야 하나요?"
- "실수로 터미널 창을 닫았더니 돌리던 서버가 같이 꺼져버렸어요!"

**해결책:**
- **`&`** 를 붙여서 백그라운드로 보내면 터미널을 계속 쓸 수 있습니다.
- **`nohup`** 을 쓰면 내가 로그아웃하고 집에 가도 서버는 계속 돌아갑니다.

---
## Code Core Points: ① 백그라운드 실행 (`&`)

명령어 맨 뒤에 **`&` (앰퍼샌드)**만 붙이면 됩니다.

```bash
# 1. 그냥 실행 (Foreground)
# 100초 동안 아무것도 못함
sleep 100

# 2. 백그라운드 실행 (&)
# 실행 즉시 프롬프트($)가 떨어져서 딴짓 가능
sleep 100 & 
# 결과: [1] 12345 (Job번호와 PID를 알려줌)
```

---
## Code Core Points: ② 불사신 만들기 (`nohup`) 

`&`만 쓰면 터미널 닫을 때 같이 죽습니다. 
**퇴근하려면 `nohup`이 필수**입니다.

문법 : `nohup [명령어] &`

```bash
# "터미널이 끊겨도(HUP 시그널) 무시하고 계속 돌아라"
nohup python my_server.py &

# 결과:
# nohup: ignoring input and appending output to 'nohup.out'
# (출력 내용은 자동으로 nohup.out 파일에 저장됨)
```

---
## Code Core Points: ③ 작업 관리 (`jobs`, `fg`, `bg`)

이미 실행 중인 녀석들을 관리하는 방법입니다.

```bash
# 1. 현재 실행 중인 작업 목록 보기
jobs
# [1]+  Running   nohup python app.py &

# 2. 백그라운드 녀석을 다시 앞으로 데려오기 (Foreground)
fg %1  # ([1]번 작업을 데려옴)

# 3. 실행 중인 녀석을 잠시 멈추기 (Suspend)
# (키보드 단축키)
Ctrl + Z 

# 4. 멈춘 녀석을 뒤에서 다시 돌리기 (Background)
bg %1
```

---
## 초보자가 자주 하는 실수 (Misconceptions)

### ① "`nohup` 했는데 로그가 안 보여요!"

- `nohup`은 기본적으로 화면 출력을 **`nohup.out`** 파일에 저장합니다.
- 실시간으로 보고 싶다면: `tail -f nohup.out`

### ② "이미 실행시킨 걸 `nohup`으로 바꿀 수 있나요?"

- 이미 돌고 있는 건 `nohup`을 못 붙입니다. 대신 **`disown`** 을 쓰면 됩니다.
    
    1. `Ctrl + Z` (일시 정지)
    2. `bg` (백그라운드 실행)
    3. `disown -h` (족보에서 파서 터미널과 연 끊기)

### ③ "백그라운드 프로세스는 어떻게 끄나요?"

- `Ctrl + C`가 안 먹힙니다.
- **`jobs`** 나 **`ps`** 로 PID를 찾아서 **`kill`** 명령어로 죽여야 합니다.
    - `kill -9 [PID]`



