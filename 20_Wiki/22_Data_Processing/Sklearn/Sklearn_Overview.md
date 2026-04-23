---
aliases:
  - sklearn
  - numpy
  - pandas
tags:
  - Sklearn
related:
  - "[[00_Skelen_HomePage]]"
---
# Sklearn 개요

## sklearn이란

- 파이썬 머신러닝 라이브러리
- 전처리 / 모델 학습 / 평가까지 한 패키지에서 해결
- numpy, pandas와 함께 데이터 분석 3대장

```python
import sklearn
print(sklearn.__version__)
```

---

## 핵심 모듈 구조

|모듈|역할|예시|
|---|---|---|
|`sklearn.feature_extraction`|텍스트 → 숫자 변환|TfidfVectorizer|
|`sklearn.metrics.pairwise`|유사도 계산|cosine_similarity|
|`sklearn.preprocessing`|데이터 정규화|StandardScaler|
|`sklearn.model_selection`|학습/테스트 분리|train_test_split|
|`sklearn.metrics`|모델 성능 평가|accuracy_score|
|`sklearn.pipeline`|전처리+모델 묶기|Pipeline|

---

## fit · transform · predict 패턴

sklearn의 모든 클래스는 이 3가지 메서드를 따라요.

```python
from sklearn.preprocessing import StandardScaler

scaler = StandardScaler()

# fit: 데이터 통계(평균, 표준편차) 학습
scaler.fit(X_train)

# transform: 학습한 기준으로 데이터 변환
X_scaled = scaler.transform(X_train)

# fit_transform: fit + transform 한번에 (학습 데이터에만 사용)
X_scaled = scaler.fit_transform(X_train)

# ⚠️ 테스트 데이터는 transform만! fit 다시 하면 안 됨
X_test_scaled = scaler.transform(X_test)
```

---
