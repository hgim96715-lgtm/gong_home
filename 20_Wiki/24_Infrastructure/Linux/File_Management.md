---
aliases:
  - 파일관리
  - mkdir
  - touch
  - cp
  - mv
  - rm
  - echo
tags:
  - Linux
related:
  - "[[File_Navigation]]"
  - "[[00_Linux_HomePage]]"
---
##  개념 한 줄 요약

**"리눅스의 호흡과도 같은 파일/폴더 생성, 복사, 이동, 삭제 4대장."**

---
## 이것이 왜 필요한가 (Why)

**문제:**
- 리눅스(CLI)에는 마우스 우클릭으로 만드는 '새 폴더'도, 드래그 앤 드롭도, 실수로 지운 파일을 살려주는 '휴지통'도 없다.

**해결:**
- `echo`로 데이터를 즉석에서 만들고, `mkdir -p`로 깊은 폴더를 한 번에 파고, `rm -rf`로 찌꺼기를 치우는 법을 알아야 서버를 자유자재로 다룰 수 있다.

---
## 실무 적용 사례 (Practical Context)

데이터 엔지니어는 매일 터미널에서 살림을 합니다:

1.  **데이터 수집:** `mkdir`로 오늘 날짜 폴더(`2026-01-26`) 생성.
2.  **테스트 데이터 생성:** `echo`를 써서 가짜 데이터(Dummy Data)를 만들어 코드가 잘 도는지 확인.
3.  **데이터 분류:** 처리가 끝난 파일은 `mv`를 써서 `processed` 폴더로 이동.
4.  **디스크 정리:** 1년 지난 로그 파일은 `rm`으로 영구 삭제.

---
## Code Core Points (핵심 문법)

* **생성:** `mkdir`(폴더), `touch`(빈 파일), **`echo`**(내용 있는 파일)
* **관리:** `cp`(복사), `mv`(이동 및 이름 변경)
* **삭제:** `rm`(삭제) - **가장 주의!**

---
## 상세 코드 분석 (Detailed Analysis)

### ① `echo`: 화면 출력 & 파일 생성의 마술사 

문자열을 표준 출력(Standard Output)으로 보내는 가장 기본적인 명령어.
단순히 화면에 글자를 보여주는 것(`print`)뿐만 아니라, **파일을 만드는 가장 빠른 도구**다.

```bash
# 1. 화면에 출력하기 (기본)
echo "Hello World"
# 출력: Hello World

# 2. 파일로 저장하기 (Redirection >)
# "화면에 뿌릴 내용을 file.txt 안으로 쏴라!"
echo "user_id, name" > file.txt

# 3. 줄바꿈 문자(\n) 사용하기 (-e 옵션) 
# -e가 없으면 \n이 글자 그대로 찍힘. -e(Enable escape)를 써야 엔터키로 인식함.
echo -e "Line 1\nLine 2\nLine 3" > multi_line.txt
# 결과:
# Line 1
# Line 2
# Line 3

# 4. 기존 내용 뒤에 덧붙이기 (Append >>) 
# > (하나)는 덮어쓰기(Overwrite), >> (두 개)는 이어쓰기(Append)
echo "Line 4" >> multi_line.txt
```

### ② 파일과 폴더 만들기 (Create)

```bash
# 1. 빈 파일 만들기 (touch)
# 내용 없이 껍데기만 필요하거나, 파일의 수정 시간을 갱신할 때 씀.
touch empty_file.txt

# 2. 폴더 만들기 (mkdir)
# -p (parents): 상위 폴더가 없으면 에러 나는데, -p를 쓰면 알아서 다 만들어줌. (꿀옵션)
# 2026 >01>data
mkdir -p 2026/01/data
```

### ③ 복사하고 이동하기 (Copy & Move)

```bash
# 1. 파일 복사 (cp 원본 복사본)
cp file.txt file_backup.txt

# 2. 폴더 복사 (cp -r) 
# 폴더는 안에 내용물이 많으므로 -r (Recursive, 재귀적) 없이는 복사 불가!
cp -r 2026/01 2026/01_backup

# 3. 파일 이동 및 이름 변경 (mv)
# 폴더가 다르면 '이동', 같은 폴더면 '이름 변경'이 된다.
mv old_name.txt ./archive/   # 이동
mv file.txt new_name.txt     # 이름 변경
```


### ④ 삭제하기 (Delete) 

리눅스에는 휴지통이 없다. 엔터 치는 순간 **영구 삭제**다.


```bash
# 1. 파일 삭제
rm file.txt

# 2. 폴더 삭제 (rm -rf)
# -r: 폴더 내부까지, -f: 묻지 말고 강제로(Force)
# 실무에서 가장 신중하게 써야 하는 명령어 1위.
rm -rf ./temp_folder
```

---
## 초보자가 자주 하는 실수 (Misconceptions)

### ① "`echo` 했는데 줄바꿈이 안 돼요!"

- **이유:** 그냥 `echo "A\nB"`라고 하면 컴퓨터는 `\n`을 알파벳 글자로 인식한다.
- **해결:** 반드시 **`-e`** 옵션을 붙여야 "아, 이게 엔터키구나!"라고 알아듣는다. (`echo -e ...`)

### ② "`>` 썼다가 파일이 다 날아갔어요!" 😱

- **이유:** `>` (꺽쇠 하나)는 **"기존 내용을 싹 지우고 새로 쓰기(Overwrite)"** 다.
- **해결:** 기존 내용 밑에 추가하고 싶다면 반드시 **`>>` (꺽쇠 두 개)** 를 써야 한다.

### ③ "폴더 복사가 안 돼요 (`omitting directory`)"

- **이유:** `cp`는 기본적으로 파일만 복사한다.
- **해결:** 폴더(보따리)를 통째로 옮기려면 **`-r`** 옵션이 필수다.

