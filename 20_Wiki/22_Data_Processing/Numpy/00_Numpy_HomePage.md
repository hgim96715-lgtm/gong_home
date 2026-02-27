> [!quote] Data is the new oil, and NumPy is the refinery.
> 대규모 수치 데이터 처리를 위한 파이썬의 심장, NumPy 정복기!

---

## Core Concepts (핵심 기초)

> 파이썬 기본 List와의 차이점을 이해하고, 배열을 생성하는 기초 공사

- [[NumPy_개념과_ndarray_이해하기]] : `ndarray` 메모리 구조와 특징 (`ndarray`, `dtype`, `shape`, `List vs ndarray`, `연속 메모리`, `벡터화`)
- [[Numpy_배열_생성_메서드]] : 다양한 배열 생성법 (`np.array`, `np.zeros`, `np.ones`, `np.arange`, `np.random`, `np.eye`, `tolist`, `linspace`)

---

## Data Extraction (데이터 탐색 및 추출)

> 원하는 데이터만 쏙쏙 골라내는 기술

- [[Numpy_배열_인덱싱과_슬라이싱]] : 1D ~ 3D 배열 슬라이싱 감 잡기 (`[행, 열]`, `[start:end:step]`, `음수 인덱스`, `다차원 슬라이싱`)
- [[04_Boolean_인덱싱과_Fancy_인덱싱]] : 조건문으로 필터링하기 (`Boolean Mask`, `조건 필터링`, `Fancy Indexing`, `np.where`, `마스크 씌우기`)

---

## Shape Manipulation (배열 형태 다루기)

> 데이터 파이프라인에서 가장 에러가 많이 나는 '차원(Shape)' 맞추기 연습

- [[05_배열_형태_변경_Reshape]] : 차원 변환의 마법 (`reshape`, `flatten`, `ravel`, `-1 자동 계산`, `T 전치`, `차원 맞추기`)
- [[06_배열_합치기와_쪼개기]] : 배열 합치고 분리하기 (`concatenate`, `vstack`, `hstack`, `split`, `axis`, `차원 유지`)

---

## Operations (연산과 통계)

> for문을 버리고 NumPy의 진정한 속도를 끌어내는 벡터화 연산

- [[07_브로드캐스팅_Broadcasting]] : 모양이 다른 배열끼리의 연산 규칙 (`Broadcasting`, `차원 자동 확장`, `Shape 호환 규칙`, `(3,1) vs (1,3)`)
- [[08_수학_및_통계_함수_그리고_Axis]] : 집계 함수와 헷갈리는 axis 완벽 이해 (`sum`, `mean`, `std`, `max`, `min`, `axis=0 행방향`, `axis=1 열방향`)

---

## Trouble Shooting & Tips