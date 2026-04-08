---
aliases:
  - 리눅스 기본 명령어
  - ls
  - cd
  - pwd
  - mkdir
  - rm
  - cp
  - mv
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_File_Commands]]"
  - "[[Linux_Filesystem]]"
  - "[[Linux_Redirection]]"
---
# Linux_Basic_Commands — 기본 명령어

## 한 줄 요약

```
파일과 디렉토리를 탐색하고 관리하는 필수 명령어
리눅스를 쓰는 한 매일 사용하는 것들
```

---

---

# ① ls — 목록 보기

```bash
ls              # 현재 디렉토리 목록
ls /etc         # 특정 경로 목록
ls -l           # 상세 정보 (Long format)
ls -a           # 숨김 파일 포함 (. 으로 시작하는 파일)
ls -la          # 상세 + 숨김 파일
ls -lh          # 상세 + 파일 크기 사람이 읽기 쉽게 (KB, MB)
ls -lt          # 수정 시간 순 정렬 (최신이 위)
ls -ltr         # 수정 시간 역순 (오래된 것이 위)
ls -R           # 하위 디렉토리 재귀 출력
```

## ls -l 출력 해석 ⭐️

```bash
ls -l
# -rw-r--r-- 1 user group 1234 Apr 4 10:00 file.txt
# drwxr-xr-x 2 user group 4096 Apr 4 09:00 mydir/
```

```
①          ② ③    ④    ⑤     ⑥    ⑦       ⑧
-rw-r--r-- 1 user group 1234 Apr 4 10:00 file.txt

①  파일 타입 + 권한
   첫 글자: - 일반 파일 / d 디렉토리 / l 심볼릭 링크
   이후 9자리: rwxrwxrwx (소유자/그룹/기타 권한)

②  하드링크 수

③  소유자 (user)

④  그룹 (group)

⑤  파일 크기 (바이트)

⑥  마지막 수정 날짜

⑦  마지막 수정 시간

⑧  파일/디렉토리 이름
```

```
권한 읽는 법:
  r = read     읽기
  w = write    쓰기
  x = execute  실행
  - = 권한 없음

  rw-r--r--
  ↑↑↑↑↑↑↑↑↑
  소유자: rw- (읽기+쓰기)
  그룹:   r-- (읽기만)
  기타:   r-- (읽기만)
```

---

---

# ② cd — 디렉토리 이동

```bash
cd /etc          # 절대 경로로 이동
cd logs          # 상대 경로로 이동 (현재 위치 기준)
cd ..            # 한 단계 위로
cd ../..         # 두 단계 위로
cd ~             # 홈 디렉토리로
cd -             # 이전 디렉토리로 (뒤로 가기)
cd               # cd 만 치면 홈으로
```

```
절대 경로 vs 상대 경로:
  /etc/nginx       ← 절대 경로 (/ 로 시작)
  ../nginx         ← 상대 경로 (현재 위치 기준)

특수 경로:
  .    현재 디렉토리
  ..   부모 디렉토리
  ~    홈 디렉토리 (/home/사용자명)
  -    바로 이전 디렉토리
```

---

---

# ③ pwd — 현재 위치 확인

```bash
pwd              # 현재 절대 경로 출력
# /home/user/projects
```

```
pwd = Print Working Directory
길을 잃었을 때 현재 위치 확인
스크립트에서 현재 경로를 변수로 저장할 때도 사용
```

```bash
# 스크립트에서 활용
CURRENT_DIR=$(pwd)
echo "현재 위치: $CURRENT_DIR"
```

---

---

# ④ mkdir — 디렉토리 생성

```bash
mkdir mydir              # 디렉토리 생성
mkdir -p a/b/c           # 중간 경로 없어도 한 번에 생성 ⭐️
mkdir -m 755 mydir       # 권한 지정하며 생성
```

```
-p 옵션이 중요한 이유:
  mkdir a/b/c
  → a 없으면 에러!

  mkdir -p a/b/c
  → a, b, c 전부 없어도 한 번에 생성
  → 이미 있어도 에러 없음
```

```bash
# 여러 개 동시 생성
mkdir dir1 dir2 dir3

# 실전: 프로젝트 구조 한 번에
mkdir -p project/{src,tests,docs,logs}
# project/src / project/tests / project/docs / project/logs 생성
```

---

---

# ⑤ rm — 삭제 ⚠️

```bash
rm file.txt              # 파일 삭제
rm -i file.txt           # 삭제 전 확인 (interactive)
rm -f file.txt           # 강제 삭제 (에러 무시)
rm -r mydir              # 디렉토리 + 내용 전체 삭제 (recursive)
rm -rf mydir             # 강제 + 재귀 삭제 ← 위험!
```

```
⚠️ rm 은 휴지통 없음
   삭제하면 복구 불가
   -rf 는 신중하게

rm -rf / 하면 전체 시스템 날아감
rm -rf ./ 도 현재 디렉토리 전체 삭제
→ 경로 반드시 확인 후 실행
```

```bash
# 안전하게 쓰는 법
rm -i file.txt           # 항상 확인
ls mydir                 # 삭제 전 내용 확인
rm -ri mydir             # 디렉토리도 확인하면서 삭제
```

---

---

# ⑥ cp — 복사

```bash
cp file.txt backup.txt        # 파일 복사
cp file.txt /tmp/             # 다른 경로로 복사
cp -r mydir/ backup_dir/      # 디렉토리 전체 복사 (recursive)
cp -i file.txt dest.txt       # 덮어쓰기 전 확인
cp -u file.txt dest.txt       # 소스가 더 최신일 때만 복사
cp -p file.txt dest.txt       # 권한 / 타임스탬프 유지
```

```
파일 복사는 그냥 cp
디렉토리 복사는 반드시 -r 필요
  cp mydir/ backup/   → 에러!
  cp -r mydir/ backup/ → 정상
```

---

---

# ⑦ mv — 이동 / 이름 변경

```bash
mv file.txt /tmp/             # 파일 이동
mv old_name.txt new_name.txt  # 이름 변경 (같은 디렉토리)
mv mydir/ /tmp/               # 디렉토리 이동
mv -i file.txt dest.txt       # 덮어쓰기 전 확인
```

```
mv = 이동 + 이름 변경 둘 다
  같은 디렉토리 안에서 mv = 이름 변경
  다른 디렉토리로 mv = 이동

cp 와 달리 -r 없어도 디렉토리 이동 가능
```

---

---

# ⑧ man — 명령어 매뉴얼

```bash
man ls               # ls 매뉴얼
man cp               # cp 매뉴얼
man man              # man 자체 매뉴얼

# 매뉴얼 안에서
q          # 종료
/검색어    # 검색
n          # 다음 검색 결과
Space      # 다음 페이지
b          # 이전 페이지
```

```
man 보기 불편하면:
  ls --help       # 간단한 도움말
  tldr ls         # 예시 중심 요약 (설치 필요)
```

---

---

# ⑨ touch — 파일 생성 / 타임스탬프 갱신

```
touch 역할:
  파일 없음  → 빈 파일 새로 생성
  파일 있음  → 마지막 수정 시간 현재로 갱신 (내용 안 바뀜)
```

```bash
touch 파일명           # 빈 파일 생성 / 타임스탬프 갱신
touch a.txt b.txt c.txt  # 여러 파일 한 번에 생성
touch -t 202401010000 파일명  # 타임스탬프 직접 지정 (YYYYMMDDhhmm)
```

```bash
# 실전 활용

# 테스트용 빈 파일 빠르게 생성
touch source_file
ls -i source_file

# ls -lt 테스트 (타임스탬프 갱신)
touch file.txt
ls -lt             # file.txt 가 맨 위로 (최신)

# 빈 로그 파일 미리 만들기
touch app.log error.log
```

```
mkdir vs touch:
  mkdir → 디렉토리 생성
  touch → 파일 생성

echo "내용" > 파일 vs touch:
  echo > 파일   → 내용 있는 파일 생성
  touch 파일    → 내용 없는 빈 파일 생성
```

---

---

# ⑩ inode — 파일 고유 식별 번호 ⭐️

```
inode = 파일 시스템에서 파일의 고유 번호
  파일 이름이 아닌 inode 로 파일 실체 식별
  파일 크기 / 권한 / 소유자 / 데이터 위치 등 메타데이터 저장

확인:
  ls -i        → inode 번호 표시
  ls -il       → inode + 상세 정보
```

```bash
ls -i source_file
# 131081 source_file  ← 131081 이 inode 번호
```

---

---

# ⑪ ln — 링크 생성

```
링크 = 파일을 가리키는 또 다른 이름
  하드 링크   → 같은 inode 공유 (원본과 완전히 동일)
  심볼릭 링크 → 경로(이름)를 가리키는 포인터
```

## 심볼릭 링크 (Symbolic Link) — ln -s

```bash
ln -s 원본파일 링크파일
ln -s filea fileb

ls -il file[ab]
# 131075 -rw-r--r-- 1 user user 14 Apr 6 filea
# 131076 lrwxrwxrwx 1 user user  5 Apr 6 fileb -> filea
#   ↑다른 inode              ↑ l = 링크 / 화살표로 원본 표시
```

```
특징:
  원본과 다른 inode (별개의 파일)
  ls -l 에서 l 로 표시 + "fileb -> filea"
  원본 삭제 → 링크 깨짐 (dangling link)
  원본 재생성 → 같은 이름이면 다시 연결됨
```

## 하드 링크 (Hard Link) — ln

```bash
ln 원본파일 링크파일
ln filea filec

ls -il file[ac]
# 131075 -rw-r--r-- 2 user user 14 Apr 6 filea   ← 링크 카운트 2
# 131075 -rw-r--r-- 2 user user 14 Apr 6 filec   ← 같은 inode!
#   ↑같은 inode
```

```
특징:
  원본과 같은 inode (데이터 공유)
  ls -l 에서 일반 파일(-) 로 표시
  링크 카운트 증가 (권한 뒤 숫자)
  원본 삭제 → 데이터 유지 (inode 는 살아있음)
  원본 재생성 → 새 inode → 기존 하드링크와 연결 안 됨
```

## 심볼릭 vs 하드 링크 비교 ⭐️

|구분|심볼릭 링크|하드 링크|
|---|---|---|
|inode|다름 (별개)|같음 (공유)|
|ls 표시|`l` + `->`|`-` (일반 파일)|
|원본 삭제 시|깨진 링크|데이터 유지|
|원본 재생성 시|다시 연결됨|새 inode → 연결 안 됨|
|다른 파일시스템|가능|불가|
|디렉토리 링크|가능|불가|
|따라가는 것|이름(경로)|데이터(inode)|

```
심볼릭 → "이름"을 따라감
하드링크 → "데이터(inode)"를 따라감
```

## 원본 삭제 후 동작 차이

```bash
# 원본 삭제
rm filea

ls -l file[b-c]
# lrwxrwxrwx  fileb -> filea   ← 깨진 링크 (원본 없음)
# -rw-r--r--  filec             ← 정상 (inode 살아있음)

cat fileb   # 에러: No such file or directory
cat filec   # 정상 출력 (데이터 유지)

# 원본 재생성
vi filea    # 새 내용으로 생성 (새 inode 할당)
ls -il file[a-c]
# 131080 filea    ← 새 inode
# 131076 fileb -> filea  ← 새 원본과 다시 연결됨
# 131075 filec    ← 과거 inode (기존 데이터)

cat filea   # 새 내용
cat fileb   # 새 내용 (새 원본 연결)
cat filec   # 기존 내용 (과거 inode 유지)
```

---

---

# ⑫ cp / mv 와 inode 변화 ⭐️

```
cp → 새 inode 할당 (데이터 전체 복사)
mv → inode 유지 (이름만 변경)
```

```bash
touch source_file
ls -i source_file
# 131081 source_file

# cp → 새 inode
cp source_file copied_file
ls -i source_file copied_file
# 131081 source_file   ← 원본
# 131082 copied_file   ← 새 inode (다른 번호)

# mv → 같은 inode
mv source_file renamed_file
ls -i renamed_file
# 131081 renamed_file  ← inode 유지! (이름만 바뀜)

# mv 로 디렉토리 이동
mkdir target_dir
mv renamed_file target_dir/
ls -i target_dir/renamed_file
# 131081 target_dir/renamed_file  ← inode 그대로
```

```
cp:
  데이터 전체를 새로 씀
  새 inode 할당
  하드링크 있어도 연결 끊김

mv:
  파일 시스템의 이름(경로)만 수정
  inode 변하지 않음
  같은 파일시스템 안에서만 inode 유지
  (다른 파일시스템으로 mv → 사실 cp + rm)
```

## cp -i — 덮어쓰기 전 확인

```bash
cp -i copied_file target_dir/renamed_file
# cp: overwrite 'target_dir/renamed_file'? y/n
```

---

---

# 자주 쓰는 조합 패턴

```bash
# 현재 위치 확인 후 이동
pwd
cd /var/log

# 로그 파일 목록 최신순
ls -lt /var/log

# 숨김 파일 포함 상세 보기
ls -la ~

# 디렉토리 구조 한 번에 생성
mkdir -p ~/projects/my_app/{src,tests,config,logs}

# 파일 복사 후 원본 삭제 (= mv 와 동일)
cp file.txt /backup/file.txt
rm file.txt

# 삭제 전 확인
ls -la target_dir/
rm -ri target_dir/
```

---

---

# 한눈에 정리

|명령어|역할|자주 쓰는 옵션|
|---|---|---|
|`ls`|목록 보기|`-la` (상세+숨김) / `-lh` (크기) / `-lt` (시간순)|
|`cd`|이동|`..` (상위) / `~` (홈) / `-` (이전)|
|`pwd`|현재 위치|-|
|`mkdir`|디렉토리 생성|`-p` (중간 경로 자동 생성)|
|`rm`|삭제|`-r` (디렉토리) / `-i` (확인) / `-f` (강제)|
|`cp`|복사|`-r` (디렉토리) / `-i` (확인) / `-p` (속성 유지)|
|`mv`|이동/이름변경|`-i` (확인)|
|`touch`|빈 파일 생성 / 타임스탬프 갱신|`-t` (시간 지정)|
|`man`|매뉴얼|`q` 종료 / `/` 검색|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`cp mydir/ backup/` 에러|디렉토리는 `-r` 필요|`cp -r mydir/ backup/`|
|`mkdir a/b/c` 에러|중간 경로 없음|`mkdir -p a/b/c`|
|`rm` 후 복구 불가|휴지통 없음|`rm -i` 로 확인하며 삭제|
|`ls -l` 권한 해석 못함|rwx 의미 모름|`r=읽기 w=쓰기 x=실행`|
|`cd` 후 어디 있는지 모름||`pwd` 로 현재 위치 확인|
|심볼릭 링크 깨짐|원본 삭제됨|원본 유지 or 하드 링크 사용|
|하드 링크인데 재생성 후 연결 안 됨|새 inode 할당됨|심볼릭은 재연결 / 하드는 과거 데이터 유지|
|`cp` 후 inode 바뀜|정상 동작|`cp`=새 inode / `mv`=inode 유지|