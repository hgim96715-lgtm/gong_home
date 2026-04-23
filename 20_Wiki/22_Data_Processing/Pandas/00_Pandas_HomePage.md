
## 핵심 원칙

```python
# ❌ for 문은 판다스의 적 — 매우 느림
for i, row in df.iterrows():
    df.loc[i, "new_col"] = row["col"] * 2

# ✅ 벡터화 연산 — 빠름
df["new_col"] = df["col"] * 2
```

---

---

## Level 1. 시작하기 (Setup & I/O)

```
일단 데이터를 불러와야 요리를 하죠.
```

|노트|핵심 개념|
|---|---|
|[[Pandas_DataStructures]]|`Series` / `DataFrame` / `1차원 vs 2차원` / `dtype` / `index`|
|[[Pandas_DataFrame_Basics]]|생성 / 선택 / 수정 / 저장 / 핵심 패턴 모음|
|[[Pandas_Read_Write]]|`read_csv` / `to_parquet` / `read_json` / `to_csv` / `encoding`|

---

---

## Level 2. 데이터 훑어보기 (Inspection)

```
데이터가 어떻게 생겼는지 엑셀처럼 열어봐요.
```

|노트|핵심 개념|
|---|---|
|[[Pandas_Inspection]]|`head` / `tail` / `info` / `describe` / `shape` / `sample` / `nunique`|
|[[Pandas_Selection]]|`loc` / `iloc` / `columns` / `[]` / `[[]]` / 행 번호 vs 레이블|
|[[Pandas_Datatypes_Filtering]]|`select_dtypes` / `include` / `exclude` / `number` / `object`|

---

---

## Level 3. 데이터 다듬기 (Cleaning)

```
더러운 데이터를 깨끗하게 씻겨줘요.
```

|노트|핵심 개념|
|---|---|
|[[Pandas_Filtering]]|`Boolean Indexing` / `&` / `\|` / `~` / `isin` / `query`|
|[[Pandas_Missing_Values]]|`fillna` / `dropna` / `isna` / `notna` / `ffill` / `bfill`|
|[[Pandas_Type_Conversion]]|`astype` / `to_datetime` / `to_numeric` / `errors='coerce'`|

---

---

## Level 4. 모양 바꾸기 (Transformation)

```
데이터를 자르고 붙여서 원하는 형태로 만들어요.
```

|노트|핵심 개념|
|---|---|
|[[Pandas_Column_Operations]]|`drop` / `rename` / `assign` / `columns` / `axis=1`|
|[[Pandas_String_Methods]]|`.str 접근자` / `str.split` / `str.contains` / `str.replace` / `str.strip`|
|[[Pandas_Json_Normalize]]|`json_normalize` / `record_path` / `meta` / 중첩 JSON / API 처리|

---

---

## Level 5. 합치기와 요약 (Aggregation)

```
여러 데이터를 하나로 모으고 통계를 내요.
```

|노트|핵심 개념|
|---|---|
|[[Pandas_Merge_Concat]]|`merge` / `concat` / `how` / `on` / `left·right·inner·outer` / `axis`|
|[[Pandas_Groupby]]|`groupby` / `agg` / `transform` / `size` / `reset_index`|
|[[Pandas_Pivot]]|`pivot_table` / `melt` / `values` / `index` / `columns` / `aggfunc`|

---

---

## Level 6. 고급 기능 (Advanced)

```
반복문 없이 고속으로 처리해요.
```

|노트|핵심 개념|
|---|---|
|[[Pandas_Apply_Map]]|`apply` / `map` / `lambda` / `axis=0/1` / Vectorization|
|[[Pandas_Efficiency]]|`Parquet` / `chunksize` / `dtype 최적화` / `category` / 메모리 절약|

---

---

## Level 7. 시각화 (Visualization)

```
분석한 데이터를 그래프로 보여줘요.
```

|노트|핵심 개념|
|---|---|
|[[Pandas_Plotting]]|`plot` / `bar` / `line` / `hist` / `scatter` / `figsize` / matplotlib 연동|

---

---

## 관련 노트

|노트|연결 이유|
|---|---|
|[[Numpy_Array_Basics]]|Pandas 내부가 NumPy 배열로 동작|
|[[Python_List_Comprehension]]|DataFrame 컬럼 선택 / 필터링에 활용|
|[[Python_Lambda_Map]]|apply + lambda 패턴|
|[[Python_Dictionaries]]|DataFrame 생성 / json_normalize|

---

---
