---
aliases:
  - tar
  - gzip
  - 압축
  - 파일압축
  - 백업
tags:
  - Linux
related:
  - "[[SSH_Connection]]"
  - "[[File_Management]]"
  - "[[00_Linux_HomePage]]"
---
## 개념 한 줄 요약

**"리눅스의 '이사짐 싸기' 세트"**

* **`tar` (테이프):** 흩어진 물건(파일)들을 박스 하나에 **담는(묶는)** 역할. (용량은 안 줄어듦)
* **`gzip` (압축기):** 박스의 공기를 빼서 **부피를 줄이는(압축)** 역할.
* **`tar.gz`:** 물건을 박스에 담고(`tar`) + 공기까지 뺀(`gzip`) 상태. 실무에서 99% 이 포맷을 쓴다.

---
## 왜 필요한가? (Why)

**문제점:**
- 로그 파일 1,000개를 친구에게 보내야 한다. 파일 1,000개를 하나씩 전송할 것인가?
- 10GB짜리 텍스트 데이터를 백업해야 하는데, 하드디스크 용량이 부족하다.

**해결책:**
- **`tar`** 로 1,000개 파일을 `logs.tar` 파일 하나로 뭉친다. (전송 편리)
- **`gzip`** 으로 10GB를 2GB로 줄여서 저장한다. (공간 절약)

---

## 실무 적용 사례 (Practical Context)

1.  **로그 파일 백업:**
    - "어제치 로그 파일들(`*.log`) 싹 묶어서 압축해놔." -> `tar -czvf logs.tar.gz *.log`
2.  **프로젝트 배포:**
    - 내 컴퓨터에서 짠 소스 코드를 서버로 보낼 때, 폴더째로 압축해서 전송(`scp`)하고 서버에서 푼다.
3.  **도커 이미지 관리:**
    - 도커 이미지(`docker save`)를 파일로 내보낼 때 내부적으로 `tar`로 묶여서 나온다.

---

## Code Core Points (핵심 명령어)

옵션 순서는 상관없지만, **`-f`는 반드시 파일 이름 바로 앞**에 와야 한다. 
(Tip: '묶을 땐 `cvf`, 압축할 땐 `czvf`, 풀 땐 `xzvf`' 공식처럼 외우자.)

### A. 압축하기 (묶기) 

**주문:** `czvf` (창조하고, 지집으로, 보여주며, 파일로!)

```bash
# 문법: tar -czvf [생성할파일.tar.gz] [압축할폴더_또는_파일]
# -c: Create (새로 만듦)
# -z: Gzip (압축까지 함)
# -v: Verbose (과정 보여줌)
# -f: File (파일 이름 지정) - ⭐️ 항상 맨 뒤에 와야 함!

# "file1과 file2를 합쳐서 archive.tar로 만들어라" 
tar -cvf archive.tar file1.txt file2.txt

# "압축까지 해서 archive.tar.gz로 만들어라" 
tar -czvf archive.tar.gz file1.txt file2.txt

tar -czvf project_backup.tar.gz ./my_project
```

### B. 압축 풀기 (해제) 

**주문:** `xzvf` (해제하고, 지집으로, 보여주며, 파일로!)

```bash
# 문법: tar -xzvf [풀_파일.tar.gz]
# -x: Extract (꺼냄/해제)
tar -xzvf project_backup.tar.gz

# 옵션: x(Extract), z(Gzip 해제) 
# "archive.tar.gz를 현재 폴더에 풀어라" 
tar -xzvf archive.tar.gz
```

### C. 그냥 파일 하나만 압축할 때 (`gzip`)

폴더가 아니라 파일 하나만 가볍게 줄이고 싶을 때 쓴다.

```bash
# 1. 압축하기 (원본 파일이 사라지고 .gz가 생김)
gzip large_file.txt
# 결과: large_file.txt.gz 생성됨

# 두개 개별 압축
# 두 파일이 하나로 합쳐지지 않고, 각각 따로따로 압축됩니다
gzip file1.txt file2.txt

# 2. 풀기 (gunzip = gzip -d)
gunzip large_file.txt.gz
# 또는
gzip -d large_file.txt.gz

# 개별로 풀기
gunzip file1.txt.gz file2.txt.gz
```


#### 안에 확인하는 방법 

1. 박스를 뜯지 않고(압축을 풀지 않고) 안에 뭐 들었는지 **목록(List)** 만 보고 싶을 땐 **`-t`** 옵션을 씁니다.

```bash
# 옵션: t (List, 목록 보기), f (File)
# "archive.tar.gz 안에 뭐 들었는지 리스트만 보여줘"
tar -tf archive.tar.gz
```

2. 진짜로 들어가려면 

```bash
# 1단계: 짐을 풀 '방(폴더)'을 먼저 만듭니다. (깔끔하게 정리하기 위해)
mkdir my_data

# 2단계: 박스를 뜯어서 그 방에 내용물을 쏟아붓습니다.
# -C (Change directory): "이 폴더에다가 풀어라!"
tar -xzvf archive.tar.gz -C ./my_data

# 3단계: 이제 방이 생겼으니 들어갈 수 있습니다!
cd my_data
```
