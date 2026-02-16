
## Level 1. 시작하기 (Setup & I/O)

> "일단 데이터를 불러와야 요리를 하죠."
> 
- [[Pandas_DataStructures]] : Series와 DataFrame의 차이(1차원 vs 2차원), Schema란?(설계도) 
- [[Pandas_Read_Write]] : 파일 읽고 쓰기 (`read_csv`, `to_parquet`, `read_json`)

## Level 2. 데이터 훑어보기 (Inspection)

> "데이터가 어떻게 생겼는지 엑셀처럼 열어봐요."
- [[Pandas_Inspection]] : 데이터 요약과 정보 확인 (`head`, `info`, `describe`, `shape`,`tail`,`sample`)
- [[Pandas_Selection]] : 원하는 행/열만 쏙 골라내기 (`loc`, `iloc`, `columns`)
- [[Pandas_Datatypes_Filtering]] :  특정 타입(숫자, 문자 등)만 골라내기 (`select_dtypes`)

## Level 3. 데이터 다듬기 (Cleaning)

> "더러운 데이터를 깨끗하게 씻겨줘요."

- [[Pandas_Filtering]] : 조건에 맞는 데이터만 남기기 (Boolean Indexing)
- [[Pandas_Missing_Values]] : 구멍 난 데이터 채우거나 버리기 (`fillna`, `dropna`, `isna`)
- [[Pandas_Type_Conversion]] : 숫자/날짜 형식 맞추기 (`astype`, `to_datetime`)

## Level 4. 모양 바꾸기 (Transformation)

> "데이터를 자르고 붙여서 원하는 형태로 만들어요."

- [[Pandas_Column_Operations]] : 컬럼 추가, 삭제, 이름 변경 (`drop`, `rename`)
- [[Pandas_String_Methods]] : 문자열 쪼개고 합치기 (`.str` 접근자)
- [[Pandas_Json_Normalize]] : 꼬불꼬불한 JSON 펴기 (API 처리 필수!) 

##  Level 5. 합치기와 요약 (Aggregation)

> "여러 데이터를 하나로 모으고 통계를 내요."

- [[Pandas_Merge_Concat]] : SQL Join처럼 테이블 합치기 (`merge`, `concat`)
- [[Pandas_Groupby]] : 그룹별로 묶어서 계산하기 (`groupby`, `agg`)
- [[Pandas_Pivot]] : 피벗 테이블 만들기 (`pivot_table`, `melt`)

## Level 6. 고급 기능 (Advanced)

> "반복문 없이 고속으로 처리해요."

- [[Pandas_Apply_Map]] : 함수를 데이터 전체에 적용하기 (`apply`, `lambda`)
- [[Pandas_Efficiency]] : 대용량 데이터 처리 팁 (Parquet, Chunksize)

## **Level 7. 시각화 및 대시보드 (Visualization)** 

>"분석한 데이터를 그래프나 표로 예쁘게 보여줘요."

- [[Pandas_Plotting]] : 판다스 내장 그래프 그리기 (`plot`, `bar`, `line`)



---

* **"반복문(for) 써도 되나요?"**: 판다스에서 `for`문을 쓰면 매우 느려집니다. 
* 가능한 `apply`나 판다스 내장 함수(Vectorization)를 쓰는 습관을 들여야 합니다.