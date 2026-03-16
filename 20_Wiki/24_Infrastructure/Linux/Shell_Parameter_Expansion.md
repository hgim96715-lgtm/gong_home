---
aliases:
  - 파라미터 확장
  - Bash 대소문자 변환
  - 변수 치환
  - ${}
tags:
  - Linux
related:
  - "[[Linux_Shell_Script]]"
  - "[[Environment_Variables]]"
  - "[[Linux_Text_Transformation_tr]]"
  - "[[Linux_Shell_Script]]"
  - "[[00_Linux_HomePage(기존)]]"
---
## 한줄 요약

**Parameter Expansion (파라미터 확장)** 은 변수의 값을 단순히 꺼내는(`$VAR`) 것을 넘어, **꺼내면서 동시에 가공(변환, 자르기, 기본값 설정)** 하는 Bash의 강력한 내부 기능입니다. 
`${...}` 중괄호 문법을 사용합니다.

---
##  `tr`이 있는데 왜 이걸 쓰나?

* **속도(Performance):** `tr`, `sed`, `awk`는 **외부 프로그램**입니다. 
* 이걸 실행하려면 리눅스는 프로세스를 새로 만들어야(fork) 합니다. 
* 반면 `${}`는 Bash가 메모리에서 **직접 처리(Built-in)** 하므로 훨씬 빠릅니다.
* **간결함:** 파이프(`|`)를 연결할 필요 없이 변수 자체에서 해결되므로 코드가 짧아집니다.

---
##  Code Core Points

### 대소문자 변환 (Bash 4.0 이상)

* **`{bash}${var,,}`**: 모든 문자를 **소문자**로 (Lowercase)
* **`${var^^}`**: 모든 문자를 **대문자**로 (Uppercase)

```bash
# 예시
name="HeLLo"
echo ${name,,}  # 결과: hello
echo ${name^^}  # 결과: HELLO
```

---
## 초보자들이 많이 하는 실수!

**"모든 리눅스에서 되나요?"** ❌

- 이 문법(`,,`, `^^`)은 **Bash 4.0 버전 이상**에서만 작동합니다. 
- 아주 오래된 서버나, Bash가 아닌 `sh`만 있는 환경에서는 에러가 납니다. (그럴 땐 `tr`을 써야 함)

**"원본이 바뀌나요?"**

- `echo ${c,,}`를 했다고 해서 변수 `c` 자체가 소문자로 영구히 바뀌는 건 아닙니다.
- 바꾸고 싶다면 `c=${c,,}` 처럼 다시 대입해야 합니다.