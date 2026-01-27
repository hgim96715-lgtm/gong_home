---
aliases:
  - uptime
  - 업타임
  - 부하량
  - 서버상태
tags:
  - Linux
related:
  - "[[Process_Management]]"
  - "[[Disk_Management]]"
---
## 개념 한 줄 요약

**"서버가 마지막 재부팅 후 얼마나 오래 깨어있었는지, 그리고 지금 얼마나 힘들어하는지(부하) 보여주는 명령어."** 

---
##  왜 필요한가? (Why)

**문제점:**
- "서버가 느려졌는데, 지금 갑자기 느려진 건가? 아니면 아까부터 계속 힘들었던 건가?"
- "이 서버, 언제 마지막으로 껐다 켰지? 혹시 몇 달 동안 방치된 건 아닌가?" 

**해결책:**
- [cite_start]**`uptime`** 한 방으로 **서버 가동 시간(Stability)**과 **최근 1분, 5분, 15분간의 부하량(Load Average)**을 즉시 확인한다. 

---

## 3. 실무 적용 사례 (Practical Context)

엔지니어가 터미널에 접속하자마자 습관적으로 치는 명령어 1순위입니다.

1.  [cite_start]**시스템 안정성 체크:** "Up 300 days"라고 뜨면 아주 안정적인 서버, "Up 5 min"이면 방금 재부팅(장애 발생 가능성)된 서버. [cite: 270]
2.  [cite_start]**성능 병목 초동 진단:** 사용자가 "접속이 안 돼요"라고 할 때, `uptime`을 쳐서 **Load Average**가 치솟아 있다면 CPU 과부하가 원인임을 바로 알 수 있음. [cite: 271]

---

## 4. Code Core Points: 결과 해석법 (Output Analysis)

명령어는 아주 간단합니다. 그냥 치면 됩니다.

```bash
$ uptime
# 결과 예시:
# 06:00:25 up 2 days, 9:25, 0 users, load average: 0.06, 0.02, 0.00