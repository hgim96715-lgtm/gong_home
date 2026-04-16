---
tags:
  - Linux_Test
related:
  - "[[00_Linux_Challenge_DashBoard]]"
  - "[[Linux_Disk]]"
  - "[[Linux_RAID]]"
  - "[[Linux_LVM]]"
source: Linux Journey
difficulty:
  - CompTIA Linux+ 자격증 취득 실습 랩
---
## 📌 랩 목표

Linux 환경에서 LVM(논리 볼륨 관리자)과 RAID 1(소프트웨어 RAID)을 구성하여 스토리지를 유연하게 관리하고, 데이터 중복성 및 마운트 영구화를 설정하는 방법을 학습합니다.

---

## 📖 학습 내용

---

### 🔷 섹션 1 — pvcreate 및 vgcreate 를 사용한 LVM 초기화

**개념 요약**

- **LVM(Logical Volume Manager):** Linux에서 스토리지 장치를 유연하고 강력하게 관리하기 위한 시스템입니
- **물리 볼륨 (PV, Physical Volumes):** 하드 드라이브 파티션이나 시뮬레이션 디스크(loop 장치 등)와 같은 실제 블록 장치입니다.
- **볼륨 그룹 (VG, Volume Groups):** 하나 이상의 물리 볼륨(PV)을 하나로 묶어 거대한 스토리지 풀로 만든 것입니다.
- **논리 볼륨 (LV, Logical Volumes):** 볼륨 그룹(VG)의 가용 공간을 쪼개어 만드는 "가상 파티션"입니다. 이 위에 파일 시스템을 올리게 됩니다.
- **초기화 준비:** `lvm2`와 `mdadm` 패키지가 필요하며, 가상 디스크 생성을 위해 `truncate`(지정한 크기의 파일 생성)와 `losetup`(파일을 블록 장치처럼 연결)을 사용합니다.
- **생성 및 확인:** `pvcreate`로 물리 볼륨을 초기화하고, `vgcreate`로 볼륨 그룹(예: `labvg`)을 생성합니다. 상태 확인은 `pvs`, `pvdisplay`, `vgs`, `vgdisplay` 등을 활용합니다

**명령어 / 문법**

```bash
sudo apt-get update && sudo apt-get install -y lvm2 mdadm

truncate -s 256M disk1.img disk2.img
ls -lh

sudo losetup /dev/loop20 disk1.img sudo losetup /dev/loop21 disk2.img

sudo pvcreate /dev/loop20 /dev/loop21
sudo pvs
sudo pvdisplay

sudo vgcreate labvg /dev/loop20 /dev/loop21
sudo vgs

```

**실습 메모**

- **핵심 인사이트:** LVM의 기본 구조는 **물리적 디스크(PV) ➡️ 스토리지 풀(VG) ➡️ 사용할 파티션(LV)** 순서로 만들어진다는 것을 기억하자.
- 이번 실습에서는 실제 하드디스크를 추가하는 대신 `truncate`와 `losetup`을 통해 빈 파일을 가상 드라이브(loop 디바이스)로 만들어서 사용

---

### 🔷 섹션 2 — lvcreate 및 mkfs 를 사용한 논리 볼륨 생성 및 마운트

**개념 요약**

- **논리 볼륨(LV) 생성:** 만들어둔 볼륨 그룹(`labvg`)에서 실질적인 파티션 역할을 할 LV를 생성합니다. `lvcreate` 명령에 `-L`(용량), `-n`(이름) 옵션을 주어 생성하며, `lvs`로 확인합니다.
- **파일 시스템 포맷:** 생성된 LV(`/dev/labvg/lablvm`)는 아직 데이터를 저장할 수 없는 원시(Raw) 상태입니다. `mkfs.ext4` 명령을 사용하여 Linux 표준 파일 시스템인 ext4로 포맷해야 합니다.
- **마운트(Mount):** 포맷이 완료된 장치를 사용자가 접근할 수 있는 디렉터리(`/lablvm`)에 연결하는 과정입니다. `mount` 명령을 사용하며, `df -h`로 디스크 사용량 및 마운트 상태를 점검합니다.

**명령어 / 문법**

```bash
sudo lvcreate -L 200M -n lablvm labvg
sudo lvs
sudo mkfs.ext4 /dev/labvg/lablvm

sudo mkdir /lablvm
sudo mount /dev/labvg/lablvm /lablvm
```

**실습 메모**

- "공간 할당"과 "사용 준비"는 별개다! 
- `lvcreate`로 공간을 잘라내더라도, `mkfs`로 포맷(길을 닦는 과정)을 하지 않으면 파일을 저장할 수 없다. 마지막으로 `mount`를 통해 특정 폴더에 연결해주어야만 비로소 접근이 가능하다.

---

### 🔷 섹션 3 — lvresize 를 사용한 LVM 논리 볼륨 크기 조정

**개념 요약**

- **동적 용량 확장:** LVM의 가장 큰 장점 중 하나로, 시스템 운영 중에도 볼륨의 크기를 쉽게 변경할 수 있습니다.
- **`-r` 플래그의 중요성:** `lvresize` 명령 사용 시 반드시 `-r` 플래그를 포함해야 합니다. 이 옵션은 논리 볼륨(그릇)의 크기를 늘리면서 내부의 파일 시스템(내용물)도 함께 확장시켜 줍니다. 누락할 경우 껍데기만 커지고 실제 사용 가능한 공간은 그대로인 상황이 발생합니다.
-  **크기 지정:** `-L` 옵션 뒤에 `300M` (특정 용량으로 맞춤) 또는 `+100M` (현재 용량에서 추가) 형태로 지정합니다.

**명령어 / 문법**

```bash
sudo lvresize -r -L 300M /dev/labvg/lablvm
df -h /lablvm
sudo lvresize -r -L 400M /dev/labvg/lablvm
df -h /lablvm
```

**실습 메모**

- 기존 파티션 방식에서는 용량을 늘리려면 파티션을 삭제하고 다시 만들거나 매우 복잡한 과정을 거쳐야 하는데, LVM은 명령어 한 줄로 확장이 가능하다! 특히 `-r` (resizefs) 옵션은 매우 중요. 파티션 크기만 늘리고 파일 시스템 크기를 안 늘리면 말짱 도루묵이므로 무조건 같이 쓴다고 기억하자.

----
### 🔷 섹션 4 — mdadm 을 사용한 RAID 1 어레이 구축 및 마운트

**개념 요약**

- **RAID 1 (미러링):** 두 개의 디스크에 완전히 똑같은 데이터를 동시에 기록하는 방식입니다. 하나의 디스크가 고장 나도 다른 하나에 데이터가 살아있어 안정성(중복성)이 매우 뛰어납니다.
- **소프트웨어 RAID 구성:** `mdadm` 도구를 사용하여 가상 디스크 두 개(`/dev/loop22`, `/dev/loop23`)를 `/dev/md0`라는 하나의 RAID 장치로 묶습니다.
	- `--level=1`: RAID 1 방식 지정
	- `--raid-disks=2`: 사용할 디스크 개수
- **상태 확인 및 마운트:** `/proc/mdstat` 파일을 읽어 RAID 동기화 상태를 실시간으로 확인할 수 있습니다. 구성이 완료되면 역시 `mkfs.ext4`로 포맷 후 마운트합니다.

**명령어 / 문법**

```bash
truncate -s 256M disk3.img disk4.img
sudo losetup /dev/loop22 disk3.img
sudo losetup /dev/loop23 disk4.img
sudo mdadm --create /dev/md0 --level=1 --raid-disks=2 /dev/loop22 /dev/loop23
# 진행 확인 메시지 출력 시 'y' 입력
cat /proc/mdstat
sudo mkfs.ext4 /dev/md0
sudo mkdir /labraid
sudo mount /dev/md0 /labraid
df -h /labraid
```

**실습 메모**

- LVM이 '유연성'을 위한 도구라면, RAID 1은 '안정성(백업)'을 위한 도구다. 서버 운영 시 데이터 유실을 막기 위해 필수적으로 고려되는 구성이다.
- `cat /proc/mdstat` 명령은 RAID 구성 상태나 복구 진행률을 모니터링할 때 현업에서도 수시로 사용하는 핵심 명령어이므로 꼭 외워두자.

---
### 🔷 섹션 5 — /etc/fstab 및 mdadm.conf 를 사용한 마운트 및 RAID 구성 영구화

**개념 요약**

- **설정 영구화(Persistence):** 지금까지 `mount`나 `mdadm`으로 설정한 내용은 메모리에 임시로 적용된 것이라 **재부팅하면 모두 사라집니다.**
- **RAID 설정 저장:** `mdadm --detail --scan` 출력을 `/etc/mdadm/mdadm.conf`에 추가하여 부팅 시 RAID 어레이를 올바르게 인식하도록 합니다.
- **자동 마운트 설정:** 부팅 시 자동으로 파티션이 연결되도록 `/etc/fstab` 파일에 LVM 장치와 RAID 장치의 마운트 정보를 기록합니다.
- **검증 단계:** 재부팅 전에 반드시 설정이 맞는지 테스트해야 합니다. 마운트를 해제(`umount`)한 뒤, `mount -a`(`fstab`에 있는 모든 항목 마운트)를 실행하여 에러 없이 마운트되는지 확인합니다.

**명령어 / 문법**

```bash
sudo mdadm --detail --scan
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
cat /etc/mdadm/mdadm.conf
echo '/dev/labvg/lablvm /lablvm ext4 defaults 0 0' | sudo tee -a /etc/fstab
echo '/dev/md0 /labraid ext4 defaults 0 0' | sudo tee -a /etc/fstab
tail -n 2 /etc/fstab
sudo umount /lablvm
sudo umount /labraid
sudo mount -a
df -h /lablvm /labraid
```

**실습 메모**

- 아무리 설정을 잘 해놔도 재부팅 후 날아가면 장애로 직결된다. `/etc/fstab`을 건드릴 때는 오타 하나로 시스템 부팅이 멈출 수 있으므로(Kernel Panic), 직접 재부팅하기 전에 `umount` -> `mount -a`로 정상 작동 여부를 사전에 테스트하는 것이 가장 중요!!