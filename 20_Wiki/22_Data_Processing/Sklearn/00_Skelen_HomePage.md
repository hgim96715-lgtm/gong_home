

> 목표: 텍스트 유사도 계산 · 전처리 · 모델 평가를 직접 구현할 수 있는 sklearn 실력

## 설치 & 버전 확인

```bash
pip3 install scikit-learn
python -c "import sklearn; print(sklearn.__version__)"
# 권장 버전: 1.4.x 이상
```

> 가상환경 설정 → [[Linux_Python_Env]] 참고

---

## Level 1. sklearn 기초

```
sklearn이 뭔지, 어떤 구조로 되어있는지 이해하기
```

|노트|핵심 개념|
|---|---|
|[[Sklearn_Overview]]|sklearn이란 / 주요 모듈 구조 / fit·transform·predict 패턴|
|[[Sklearn_Dataset]]|load_iris / make_classification / train_test_split|

---

## Level 2. 텍스트 처리 (NLP)

```
텍스트를 숫자로 바꿔서 비교하기
데이터 엔지니어가 가장 많이 쓰는 부분
```

| 노트                            | 핵심 개념                                                     |
| ----------------------------- | --------------------------------------------------------- |
| [[Sklearn_TF_IDF]]            | TF / IDF / TF-IDF 계산 원리 / TfidfVectorizer / fit_transform |
| [[Sklearn_CountVectorizer]]   | 단어 빈도 행렬 / vocabulary_ / get_feature_names_out            |
| [[Sklearn_Cosine_Similarity]] | 벡터 유사도 / cosine_similarity / 결과 해석 (0~1)                  |

---

## Level 3. 전처리 (Preprocessing)

```
데이터를 모델에 넣기 전에 정제하기
```

|노트|핵심 개념|
|---|---|
|[[Sklearn_StandardScaler]]|평균 0 / 표준편차 1 / fit_transform / inverse_transform|
|[[Sklearn_MinMaxScaler]]|0~1 범위로 압축 / 이상치에 민감|
|[[Sklearn_LabelEncoder]]|문자 → 숫자 / 범주형 변수 처리|
|[[Sklearn_OneHotEncoder]]|범주형 → 0/1 벡터 / get_dummies와 비교|

---

## Level 4. 모델 선택 & 평가

```
어떤 모델이 좋은지 수치로 확인하기
```

|노트|핵심 개념|
|---|---|
|[[Sklearn_Train_Test_Split]]|학습/테스트 분리 / random_state / stratify|
|[[Sklearn_Cross_Validation]]|cross_val_score / KFold / 과적합 방지|
|[[Sklearn_Metrics]]|accuracy / precision / recall / f1_score / confusion_matrix|
|[[Sklearn_Pipeline]]|Pipeline / 전처리+모델 한번에 / fit·predict|

---

## Level 5. 실전 사용 예시

```
실제 프로젝트에서 어떻게 썼는지
```

|노트|핵심 개념|
|---|---|
|[[Sklearn_NCS_ONET_Mapping]]|TF-IDF + cosine_similarity / NCS↔O*NET 유사도 매핑 실습|

---

## 관련 노트 연결

- [[Numpy]] — sklearn 내부가 numpy 배열로 동작
- [[Pandas]] — 데이터 불러오기 / 결과 저장
- [[Python_Statistics_Module]] — 기초 통계 개념