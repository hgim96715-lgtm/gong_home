---
tags:
  - Linux_Test
related:
  - "[[00_Linux_Challenge_DashBoard]]"
  - "[[Linux_Scheduling_Crontab_at]]"
source: Linux Journey
difficulty:
  - CompTIA Linux+ 자격증 취득 실습 랩
---
## 📌 랩 목표

Linux 시스템에서 예약된 작업을 관리하는 두 가지 핵심 데몬(`atd`, `cron`)과 유틸리티(`at`, `crontab`)의 사용법을 이해하고, 일회성 및 반복성 작업을 스케줄링하는 방법을 실습합니다.

---

## 📖 학습 내용

---

### 🔷 섹션 1 — 실습 환경 준비

**개념 요약**

- Linux 시스템에는 보통 `cron`이 기본 설치되어 있으나, 일회성 작업 예약 도구인 `at` 유틸리티는 `apt-get install -y at` 명령어로 수동 설치가 필요할 수 있습니다.
- 예약 명령어(`at`, `cron`)가 실제로 동작하려면 백그라운드 서비스인 데몬(`atd`, `cron`)이 실행 중이어야 합니다.
- `systemctl status` 명령어를 통해 데몬이 `active (running)` 상태인지 확인하여 스케줄러가 작업을 처리할 준비가 되었는지 점검합니다.

**명령어 / 문법**

```bash
sudo apt-get update
sudo apt-get install -y at
sudo systemctl status atd
sudo systemctl start atd
sudo systemctl status cron
```

**실습 메모**

- `systemctl`을 통해 해당 서비스가 정상적으로 실행(`active`)되고 있는지 상태를 점검하는 습관이 중요

---

### 🔷 섹션 2 — at 명령어로 일회성 작업 예약하기

**개념 요약**

- `at` 명령어는 정기적인 반복 없이, 미래의 특정 시점에 단 한 번만 실행할 스크립트나 명령어를 예약할 때 사용합니다.
- 예약 시간을 지정하면 `at>` 프롬프트가 열리며, 여기에 실행할 명령어들을 입력합니다.
- 명령어 입력을 마치고 스케줄러에 작업을 등록하려면 `Ctrl-D`를 누릅니다. 이후 작업 번호와 실행 예정 시간이 표시됩니다.

**명령어 / 문법**

```bash
at now + 1 minute
echo "The at job executed successfully." > ~/project/at_output.txt
ls -l ~/project/at_output.txt
cat ~/project/at_output.txt
rm ~/project/at_output.txt
```

**실습 메모**

- 당장 실행하기엔 서버 자원(CPU, 메모리)을 많이 쓰거나, 사용자가 적은 새벽 시간에 단발성 유지보수 스크립트를 돌려야 할 때 매우 유용하게 쓰일 수 있는 기능

---

### 🔷 섹션 3 — atq 와 atrm 으로 대기 중인 작업 관리하기

**개념 요약**

- 예약해 둔 `at` 작업들은 큐(Queue)에 대기 상태로 머물게 되며, 이를 확인하고 취소할 수 있습니다.
- `atq` (at queue) 명령어는 현재 대기 중인 모든 예약 작업의 목록과 작업 번호를 보여줍니다.
- `atrm` (at remove) 명령어 뒤에 작업 번호를 입력하면 대기 중인 특정 작업을 취소(삭제)할 수 있습니다.

**명령어 / 문법**

```bash
at now + 10 minutes
at > touch ~/project/temporary_job.txt #job 2 at Sat Apr 18 18:27:00 2026
atq
atrm <job_number>
atq
```

**실습 메모**

- 삭제(`atrm`)를 수행하기 전에는 반드시 `atq`로 목록을 먼저 조회하여 내가 삭제하려는 작업 번호가 맞는지 더블 체크하는 과정이 필수적입니다. 잘못된 번호를 입력해 다른 중요한 예약 작업을 날리는 실수를 방지

----
### 🔷 섹션 4 — crontab -e 로 반복 작업 생성하기

**개념 요약**

- `cron`은 단발성인 `at`과 달리, 정해진 일정이나 주기에 따라 지속적으로 반복 실행되어야 하는 작업을 관리합니다.
- 이러한 반복 작업 일정은 `crontab`이라는 특수 파일에 저장됩니다.
- `crontab -e` 명령어를 사용하여 텍스트 편집기를 열고 새로운 스케줄을 추가하거나 기존 일정을 수정할 수 있습니다.

**명령어 / 문법**

```bash
crontab -e
* * * * * date >> ~/project/cron_log.txt
# crontab: installing new crontab
ls -l ~/project/cron_log.txt
cat ~/project/cron_log.txt
```

**실습 메모**

- `crontab`의 시간 설정 문법인 5개의 별(`* * * * *`)이 각각 분(0-59), 시(0-23), 일(1-31), 월(1-12), 요일(0-7)을 의미한다는 것을 배웠습니다.
- 서버 운영에서 빠질 수 없는 로그 로테이션(Log Rotation), 정기 DB 백업, 시스템 상태 모니터링 스크립트 등을 자동화하는 데 있어 `cron`은 핵심적인 역할을 합니다.

---
### 🔷 섹션 5 — crontab -l 및 -r 로 Cron 작업 확인 및 삭제하기

**개념 요약**

- `crontab -l` 명령어는 현재 사용자의 활성화된 전체 cron 작업 목록을 터미널에 출력해 줍니다.
- `crontab -r` 명령어는 현재 등록된 사용자의 전체 crontab 파일을 즉시 삭제합니다.
- `-r` 옵션은 추가적인 확인 절차(Yes/No) 없이 파일 전체를 삭제해 버리므로 사용 시 각별한 주의가 필요합니다.

**명령어 / 문법**

```bash
crontab -l
crontab -r
crontab -l # no crontab for labex
rm ~/project/cron_log.txt
```

**실습 메모**

- `-l` (list)로 작업 내역을 안전하게 확인할 수 있지만, `-r` (remove)는 단일 작업이 아닌 전체 스케줄을 날려버리는 강력하고 위험한 옵션입니다.
- 실무 환경에서는 `crontab -r`을 사용하는 대신, `crontab -e`로 편집기에 들어가서 중지하고 싶은 특정 줄의 맨 앞에 `#`을 붙여 주석(Comment) 처리해 두는 것이 훨씬 안전한 관리 방법입니다. 추후에 다시 필요해질 때 주석만 해제하면 되기 때문입니다.