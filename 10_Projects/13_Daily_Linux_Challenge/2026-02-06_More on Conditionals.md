---
tags:
  - Linux_Test
status: 🟩 해결
related:
  - "[[Shell_Scripting_Basics]]"
  - "[[Redirection_Pipe]]"
  - "[[Data_Statistics]]"
source: HackerRank
---
##  문제 분석 & 파이프라인 설계 (Shell logic)

> 세 개의 정수가 주어졌을 때 (X,Y,Z) 그리고삼각형의 세 변을 나타내는 도형을 보고, 
> 그 삼각형이 부등변삼각형, 이등변삼각형 또는 정삼각형인지 판별하십시오.
>  세 변의 길이가 모두 같으면 를 출력합니다 `EQUILATERAL`.
> 그렇지 않고 양변의 두 변이 같으면 를 출력합니다 `ISOSCELES`.
> 그렇지 않으면 출력합니다 `SCALENE`.

1. **Input/Output 데이터 형태:**
    - **Input**:  세 정수(X,Y,Z)
    - **Output**:  세변이 같으면  `EQUILATERAL`, 두변이 같으면 `ISOSCELES`, 그렇지 않으면 `SCALENE`
2. **핵심 명령어 & 로직 (Pipeline):**
	- CASE 
3. **사용할 주요 옵션 (Flags):**

---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash
read X
read Y
read Z

if [ "$X" = "$Y" -a "$X" = "$Z" -a "$Y" = "$Z" ]; then
	echo "EQUILATERAL"
elif [ "$X" != "$Y" -a "$X" != "$Z" -a "$Y" != "$Z" ]; then
	echo "SCALENE"
else
	echo "ISOSCELES"
fi
```

---
## 오답 노트 & 배운 점 (Retrospective)

- **내가 실수한 부분 (Syntax/Spacing):**
	- `-a` 옵션을 사용한 조건 결합은 동작은 하지만,**가독성과 권장 방식 측면에서 `&&` 사용이 더 적절**하다는 점을 알게 되었다.
	- `{text}=` / `!=` 대신 `-eq` / `-ne` 를 사용하는 것이 더 적절함)
-  **새로 알게 된 명령어/옵션:**
	- `sort | uniq | wc -l`   파이프라인을 이용해 중복을 제거하고 서로 다른 값의 개수를 세는 방법으로,  조건문을 단순화하는 데 활용할 수 있는걸 상기시켜야겠다
	- 

---
## 더 나은 풀이 (One-liner / Efficiency)

```bash
# 숏코딩 / 효율적인 한 줄 명령어
read X
read Y
read Z

cnt=$(printf "%s\n" "$X" "$Y" "$Z" | sort | uniq | wc -l)

case "$cnt" in
    1) echo "EQUILATERAL" ;;
    2) echo "ISOSCELES" ;;
    3) echo "SCALENE" ;;
esac

```

> 세 변을 한 줄씩 출력 → 정렬 → 중복 제거 → 서로 다른 값의 개수를 세어 `cnt`에 저장