---
aliases:
  - DataFrame
  - Pandas
  - 판다스
tags:
  - Pandas
related:
  - "[[00_Pandas_HomePage]]"
  - "[[Pandas_DataStructures]]"
  - "[[Pandas_Inspection]]"
  - "[[Pandas_Selection]]"
  - "[[Pandas_Datatypes_Filtering]]"
---

# Pandas_DataFrame_Basics — DataFrame 핵심 기본

## 한 줄 요약

```
DataFrame 을 다루는 가장 자주 쓰는 기본 패턴 모음
이것만 알아도 웬만한 데이터 처리는 다 된다
```

---

---

# ① import & 환경 설정

```python
import pandas as pd
import numpy as np

# 출력 옵션 (더 많이 보기)
pd.set_option("display.max_columns", None)   # 모든 컬럼 표시
pd.set_option("display.max_rows", 100)       # 최대 100행 표시
pd.set_option("display.max_colwidth", 200)   # 컬럼 너비 확장

# 옵션 초기화
pd.reset_option("all")
```

---

---

# ② DataFrame 생성

```python
# 딕셔너리로 생성
df = pd.DataFrame({
    "Name":   ["Iron Man", "Captain", "Thor"],
    "Age":    [45, 100, 1500],
    "Weapon": ["Suit", "Shield", "Hammer"]
})

# 리스트 of 딕셔너리
df = pd.DataFrame([
    {"Name": "Iron Man", "Age": 45},
    {"Name": "Captain",  "Age": 100},
])

# CSV 읽기
df = pd.read_csv("file.csv")
df = pd.read_csv("file.csv", encoding="utf-8")
df = pd.read_csv("file.csv", encoding="cp949")  # 한글

# 복사 (원본 보호)
df2 = df.copy()
```

---

---

# ③ 기본 정보 확인

```python
df.shape          # (행 수, 열 수)
df.columns        # 컬럼 이름 목록
df.index          # 행 인덱스
df.dtypes         # 각 컬럼의 데이터 타입
df.info()         # 타입 + Non-Null + 메모리
df.describe()     # 숫자 컬럼 통계 (평균/표준편차/min/max)
df.head()         # 상위 5행
df.tail()         # 하위 5행
df.sample(5)      # 랜덤 5행
df.nunique()      # 각 컬럼의 고유값 개수
df.value_counts() # 값별 빈도 (Series)
```

---

---

# ④ 컬럼 선택 ⭐️

```python
# 단일 컬럼 → Series
df["Name"]

# 이중 대괄호 → DataFrame 유지
df[["Name", "Age"]]

# 전체 컬럼 출력
df.columns.tolist()

# 컬럼명에 조건
df[[c for c in df.columns if "date" in c.lower()]]
```

---

---

# ⑤ 행 선택 — loc / iloc ⭐️

```python
# iloc — 숫자 위치 (끝 미포함)
df.iloc[0]           # 0번째 행 (Series)
df.iloc[0:3]         # 0,1,2번째 행
df.iloc[[0, 2]]      # 0번, 2번 행

# loc — 인덱스 레이블 (끝 포함!)
df.loc[0]            # 인덱스가 0인 행
df.loc[0:2]          # 0, 1, 2 행 (끝 포함!)

# 행 + 열 동시
df.iloc[0, 1]                     # 0번행 1번열 값
df.loc[:, ["Name", "Age"]]        # 모든 행의 두 컬럼
df.iloc[0:3, 0:2]                 # 0~2행, 0~1열
```

---

---

# ⑥ 조건 필터링

```python
# 기본 조건
df[df["Age"] > 100]
df[df["Name"] == "Thor"]

# 여러 조건 (& | ~)
df[(df["Age"] > 50) & (df["Weapon"] == "Suit")]
df[(df["Age"] < 50) | (df["Age"] > 1000)]
df[~df["Name"].str.contains("Iron")]    # ~ = NOT

# isin — 여러 값 중 하나
df[df["Name"].isin(["Thor", "Captain"])]

# query — SQL 스타일 (가독성 좋음)
df.query("Age > 100")
df.query("Name == 'Thor' and Age > 1000")
```

---

---

# ⑦ 컬럼 추가 / 수정 / 삭제

```python
# 컬럼 추가
df["Power"] = df["Age"] * 10

# 조건부 컬럼 추가
import numpy as np
df["Senior"] = np.where(df["Age"] > 100, True, False)

# assign — 원본 유지하면서 새 컬럼 추가
df2 = df.assign(Power=df["Age"] * 10)

# 컬럼 이름 변경
df.rename(columns={"Name": "Hero", "Age": "Years"}, inplace=True)

# 컬럼 삭제
df.drop(columns=["Power"], inplace=True)
df.drop(columns=["Power", "Senior"])     # 여러 개

# 컬럼 순서 변경
df = df[["Name", "Weapon", "Age"]]
```

---

---

# ⑧ 결측치 처리 기본

```python
df.isna()                    # NaN 여부 (True/False)
df.isna().sum()              # 컬럼별 NaN 개수
df.dropna()                  # NaN 있는 행 전부 삭제
df.dropna(subset=["Name"])   # 특정 컬럼 기준
df.fillna(0)                 # NaN → 0
df.fillna({"Age": 0, "Name": "Unknown"})  # 컬럼별 다르게
```

---

---

# ⑨ 정렬

```python
df.sort_values("Age")                          # 오름차순
df.sort_values("Age", ascending=False)         # 내림차순
df.sort_values(["Age", "Name"], ascending=[False, True])  # 다중 정렬
df.sort_index()                                # 인덱스 기준 정렬
```

---

---

# ⑩ 그룹화 & 집계 기본

```python
# groupby + 집계
df.groupby("Weapon")["Age"].mean()
df.groupby("Weapon").agg({"Age": ["mean", "max", "count"]})

# value_counts
df["Weapon"].value_counts()
df["Weapon"].value_counts(normalize=True)  # 비율
```

---

---

# ⑪ 문자열 처리 기본 — .str 접근자

```python
df["Name"].str.upper()                   # 대문자
df["Name"].str.lower()                   # 소문자
df["Name"].str.strip()                   # 공백 제거
df["Name"].str.contains("Iron")          # 포함 여부
df["Name"].str.replace("Man", "Woman")   # 치환
df["Name"].str.split(" ")                # 분리 (리스트)
df["Name"].str.len()                     # 길이
```

---

---

# ⑫ 병합 & 연결 기본

```python
# merge (SQL JOIN)
pd.merge(df1, df2, on="ID", how="left")

# concat (위아래 붙이기)
pd.concat([df1, df2], ignore_index=True)

# concat (옆으로 붙이기)
pd.concat([df1, df2], axis=1)
```

---

---

# ⑬ 저장

```python
df.to_csv("output.csv", index=False)        # CSV 저장
df.to_parquet("output.parquet")             # Parquet 저장
df.to_json("output.json", orient="records") # JSON 저장
df.to_excel("output.xlsx", index=False)     # Excel 저장
```

---

---

# 핵심 원칙 — for문 금지 ⭐️

```python
# ❌ 느린 방식 (for + iterrows)
for i, row in df.iterrows():
    df.loc[i, "Power"] = row["Age"] * 2

# ✅ 빠른 방식 (벡터 연산)
df["Power"] = df["Age"] * 2

# ✅ 조건 처리
df["Grade"] = np.where(df["Age"] > 100, "Senior", "Junior")

# ✅ 복잡한 로직
df["Result"] = df["Age"].apply(lambda x: x * 2 if x > 100 else x)
```

```
for 문 대신:
  단순 연산    → 벡터 연산 (df["col"] * 2)
  조건 처리    → np.where
  복잡한 로직  → apply + lambda
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`df["A", "B"]` 에러|이중 대괄호 없음|`df[["A", "B"]]`|
|`df.loc[0:2]` 가 3행 반환|loc 는 끝 포함|iloc 은 끝 미포함|
|수정했는데 원본 안 바뀜|inplace=False 기본|`inplace=True` 또는 재할당|
|SettingWithCopyWarning|뷰에서 수정 시도|`.copy()` 후 수정|
|`iterrows` 너무 느림|for 문 사용|벡터 연산 / apply|