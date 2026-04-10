---
tags:
  - Linux_Test
related:
  - "[[00_Linux_Challenge_DashBoard]]"
  - "[[Linux_File_Permissions]]"
  - "[[Linux_User_Group]]"
source: Linux Journey
difficulty:
  - CompTIA Linux+ 자격증 취득 실습 랩
---
## 📌 랩 목표

Linux 환경에서 새로운 그룹을 생성하고, 사용자를 추가하며, 특정 디렉토리의 소유권(User & Group)을 변경하여 권한 체계를 관리하는 방법을 익힙니다.

---

## 📖 학습 내용

---

### 🔷 섹션 1 —  환경 준비 및 그룹 멤버십 확인

**개념 요약**

- **그룹의 목적:** 여러 사용자에게 공통된 파일 접근 권한을 한 번에 부여하기 위해 사용합니다.
- **groupadd:** 시스템에 새로운 그룹 식별자를 생성합니다.
- **usermod -aG:** 사용자를 보조 그룹(Secondary Group)에 추가합니다. `-a`(append) 옵션이 없으면 기존 그룹에서 탈퇴될 수 있으니 주의해야 합니다.

**명령어 / 문법**

```bash
sudo groupadd research            # 'research'라는 이름의 새 그룹 생성
grep 'research' /etc/group        # 그룹 생성 여부 및 GID 확인

sudo usermod -aG research labex   # labex 사용자를 research 그룹에 추가
grep 'research' /etc/group        # 그룹 멤버 목록에 labex가 포함되었는지 확인
```

**실습 메모**

-  **로그인 세션 주의:** `usermod`로 그룹에 추가된 직후에는 현재 세션에 반영되지 않습니다. 변경 사항을 적용하려면 로그아웃 후 다시 로그인하거나 `newgrp research` 명령어를 사용해야 합니다.
- **보조 그룹 vs 주 그룹:** 사용자는 하나의 주 그룹(Primary)과 여러 개의 보조 그룹(Supplementary)을 가질 수 있습니다. `-aG`는 안전하게 보조 그룹을 늘려가는 방식입니다.

---

### 🔷 섹션 2 — 디렉터리 생성 및 초기 소유권 조사

**개념 요약**

- **기본 소유권:** Linux에서 새 디렉토리를 만들면, 기본적으로 해당 디렉토리를 만든 **사용자**와 그 사용자의 **주 그룹**이 소유권을 가집니다.
- **ls -l:** 파일의 상세 정보를 보여주며, 세 번째 열은 소유자, 네 번째 열은 소유 그룹을 나타냅니다.

**명령어 / 문법**

```bash
mkdir RandD
ls -l #drwxrwxr-x 2 labex labex 6 Jun 26 10:28 RandD
```

**실습 메모**

- **디렉토리의 의미:** `ls -l` 결과 맨 앞의 `d`는 이것이 디렉토리임을 뜻합니다.
- - **생성 시점의 그룹:** 별도의 설정을 하지 않았다면 보통 자신의 사용자 이름과 동일한 그룹(UuId 기반 그룹)이 소유 그룹으로 할당됩니다.

---

### 🔷 섹션 3 — chown 명령어를 사용한 디렉터리 소유권 변경

**개념 요약**

- **chown (Change Owner):** 파일이나 디렉토리의 소유자와 그룹을 한 번에 또는 각각 변경할 수 있는 강력한 명령어입니다.
- **구문 형식:** `chown [USER]:[GROUP] [TARGET]`의 형식을 사용하며, `:`을 기준으로 앞은 사용자, 뒤는 그룹을 지정합니다.

**명령어 / 문법**

```bash
sudo chown labex:research RandD   # 소유자는 labex, 그룹은 research로 변경
```

**실습 메모**

- **재귀적 변경 (-R):** 만약 디렉토리 안에 파일이 이미 많다면 `sudo chown -R labex:research RandD`처럼 `-R` 옵션을 써서 하위 모든 파일의 소유권을 한꺼번에 바꿔야 합니다.
- **그룹만 바꾸고 싶을 때:** `chown :research RandD` (앞의 사용자 부분 생략 가능) 혹은 `chgrp research RandD` 명령어를 사용할 수도 있습니다.

----
### 🔷 섹션 4 — ls -l 명령어로 새로운 디렉터리 소유권 확인

**개념 요약**

-  **변경 결과 확인:** `chown` 작업 후 소유 그룹이 의도한 대로(research) 변경되었는지 최종 점검합니다.
- **권한의 시너지:** 이제 `research` 그룹에 속한 다른 사용자들도 이 디렉토리에 대해 그룹 권한(rwx)을 행사할 수 있게 됩니다.

**명령어 / 문법**

```bash
ls -l # drwxrwxr-x 2 labex research 6 Jun 26 10:28 RandD
```

**실습 메모**

- **협업의 기초:** 이 설정은 팀 프로젝트 시 공용 디렉토리를 만들 때 매우 유용합니다. 소유 그룹을 프로젝트 그룹으로 맞추고 그룹 권한을 `rwx`로 주면 팀원들끼리 파일을 자유롭게 공유할 수 있습니다.
- **보안 체크:** 소유권이 바뀐 후에는 `ls -l`을 통해 다른 사용자(Others)에게 불필요한 권한이 열려 있지 않은지도 함께 확인하는 습관을 들이는 것이 좋습니다

---
