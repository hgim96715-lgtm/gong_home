
> 대용량 데이터를 자유자재로 다루기 위한 Pandas 정복기!

---

## Level 1. 시작하기 (Setup & I/O)

```
일단 데이터를 불러와야 요리를 하죠.
```

|노트|핵심 개념|
|---|---|
|[[Pandas_DataStructures]]|`Series` / `DataFrame` / `1차원 vs 2차원` / `Schema` / `dtype` / `index`|
|[[Pandas_Read_Write]]|`read_csv` / `to_parquet` / `read_json` / `to_csv` / `storage_options` / `encoding`|

---

## Level 2. 데이터 훑어보기 (Inspection)

```
데이터가 어떻게 생겼는지 엑셀처럼 열어봐요.
```

|노트|핵심 개념|
|---|---|
|[[Pandas_Inspection]]|`head` / `tail` / `info` / `describe` / `shape` / `sample` / `dtypes` / `nunique`|
|[[Pandas_Selection]]|`loc` / `iloc` / `columns` / `[]` / 행 번호 vs 라벨|
|[[Pandas_Datatypes_Filtering]]|`select_dtypes` / `include` / `exclude` / `number` / `object`|

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

## Level 4. 모양 바꾸기 (Transformation)

```
데이터를 자르고 붙여서 원하는 형태로 만들어요.
```

|노트|핵심 개념|
|---|---|
|[[Pandas_Column_Operations]]|`drop` / `rename` / `assign` / `columns` / `axis=1`|
|[[Pandas_String_Methods]]|`.str 접근자` / `str.split` / `str.contains` / `str.replace` / `str.strip`|
|[[Pandas_Json_Normalize]]|`json_normalize` / `record_path` / `meta` / 중첩 JSON / API 처리 필수|

---

## Level 5. 합치기와 요약 (Aggregation)

```
여러 데이터를 하나로 모으고 통계를 내요.
```

|노트|핵심 개념|
|---|---|
|[[Pandas_Merge_Concat]]|`merge` / `concat` / `how` / `on` / `left·right·inner·outer` / `axis`|
|[[Pandas_Groupby]]|`groupby` / `agg` / `transform` / `size` / `reset_index` / named aggregation|
|[[Pandas_Pivot]]|`pivot_table` / `melt` / `values` / `index` / `columns` / `aggfunc`|

---

## Level 6. 고급 기능 (Advanced)

```
반복문 없이 고속으로 처리해요.
```

|노트|핵심 개념|
|---|---|
|[[Pandas_Apply_Map]]|`apply` / `map` / `lambda` / `axis=0/1` / Vectorization|
|[[Pandas_Efficiency]]|`Parquet` / `chunksize` / `dtype 최적화` / `category 타입` / 메모리 절약|

---

## Level 7. 시각화 (Visualization)

```
분석한 데이터를 그래프나 표로 예쁘게 보여줘요.
```

|노트|핵심 개념|
|---|---|
|[[Pandas_Plotting]]|`plot` / `bar` / `line` / `hist` / `scatter` / `figsize` / matplotlib 연동|

---

## ⚡ 핵심 원칙

> **`for`문은 판다스의 적이다.** `for`문을 쓰면 매우 느려집니다. 가능한 `apply` 나 판다스 내장 벡터화 함수를 쓰는 습관을 들이세요.

```python
# ❌ 느린 방식
for i, row in df.iterrows():
    df.loc[i, "new_col"] = row["col"] * 2

# ✅ 빠른 방식
df["new_col"] = df["col"] * 2
```

---

## 관련 노트

- [[Numpy_Array_Basics]] ← Pandas 내부가 NumPy 배열로 동작
- [[CS_Vector_Matrix]] ← DataFrame = 2차원 행렬
- [[Sklearn_TF_IDF]] ← Pandas DataFrame을 sklearn에 넘기는 패턴
- [[26_Visualization]] ← Streamlit / Superset 시각화 연동