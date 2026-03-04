---
aliases:
  - 와일드카드
  - globbing
  - 정규표현식기초
  - 별표
  - 물음표
tags:
  - Linux
  - Shell
related:
  - "[[File_Management]]"
  - "[[Linux_Filtering_Text]]"
---
## 개념 한 줄 요약

**"파일 이름을 낚아채는 '조커(Joker)' 카드."**
파일 이름이 정확히 기억나지 않거나, 수백 개의 파일을 한 번에 처리해야 할 때 사용하는 특수 기호다.

---
## 왜 필요한가? (Why)

**문제점:**
- `.log`로 끝나는 파일 100개를 지워야 하는데, `rm file1.log`, `rm file2.log`... 이렇게 100번 칠 것인가?
- "어? 파일 이름이 `data`로 시작했던 것 같은데 뒤에 날짜가 기억 안 나네."

**해결책:**
- **`rm *.log`** 한 줄이면 끝난다. (`*`이 "모든 글자"를 대신해준다)
- **`ls data*`** 치면 `data`로 시작하는 모든 파일을 찾아준다.

---
## Code Core Points: 3대장 기호

### ① `*` (Asterisk): "뭐가 오든 상관없어" (0글자 이상)

가장 많이 쓴다. 글자 수가 몇 개든, 아예 없든 다 매칭된다.

```bash
ls *.txt      # .txt로 끝나는 모든 파일 (a.txt, data.txt)
rm temp* # temp로 시작하는 모든 파일 삭제 (temporary, temp1)
cp /data/* .  # /data 폴더 안의 모든 것을 여기로 복사
```

### ② `?` (Question Mark): "딱 한 글자만"

글자 수가 정확히 맞아야 한다.

```bash
ls file?.txt
# file1.txt (O), fileA.txt (O)
# file10.txt (X) -> ?는 한 글자 자리만 채운다!
```

### ③ `[]` (Square Brackets): "이 중에서 골라"

괄호 안에 있는 글자 중 하나랑 일치하면 된다.

```bash
ls file[123].txt
# file1.txt, file2.txt, file3.txt 만 찾는다. (file4.txt는 무시)

ls [A-Z]*
# 대문자로 시작하는 모든 파일 찾기 (범위 지정 가능)
```

### ④ `^` 또는 `!` (Not): "이거 빼고"

대괄호 안에서 쓰이면 "부정(Not)"을 의미한다.

```bash
ls file[!123].txt
# file1, file2, file3을 "제외한" 나머지 (file4.txt 등)
```

---
## 초보자가 자주 하는 실수

- **`rm *` (자폭):** 아무 생각 없이 `rm *`을 치면 현재 폴더의 모든 파일이 삭제된다. 
- 항상 `ls *`로 먼저 확인하는 습관을 들이자.

