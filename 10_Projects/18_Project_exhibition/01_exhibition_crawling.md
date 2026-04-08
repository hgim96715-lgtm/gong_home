---
aliases:
  - crawl
tags:
  - Project
related:
  - "[[00_exhibition_Project]]"
  - "[[Linux_OpenSSL]]"
  - "[[Python_Web_Scraping]]"
  - "[[Python_Argparse]]"
  - "[[Python_Classes_Objects]]"
  - "[[Python_Selenium]]"
  - "[[Python_OCR]]"
---
# 🎨 인터파크 전시 ELT 프로젝트

## 📅 진행 현황

- [x] 크롤러 구현 (Selenium + EasyOCR)
- [x] PostgreSQL 적재 (15개 완료)
- [ ] dbt Transform
- [ ] Superset 대시보드
- [ ] Airflow 스케줄링

---

## 🕷️ 01_Crawling

### BeautifulSoup vs Selenium

|항목|BeautifulSoup|Selenium|
|---|---|---|
|JS 렌더링|❌|✅|
|클릭/스크롤|❌|✅|
|속도|빠름|느림|

> **인터파크 = Selenium 필수** (팝업 클릭, 무한 스크롤)

### 실행

```bash
cd ~/exhibition-data-mart/crawl
pip install -r requirements.txt

python main.py --mode test --show-browser  # 디버깅
python main.py --mode crawl --scroll 5     # 전체
python main.py --mode crawl --scroll 5 --no-ocr  # 빠르게
```

### OCR 선택: EasyOCR

|라이브러리|한글|속도|비용|
|---|---|---|---|
|Tesseract|⭐⭐⭐|느림|무료|
|**EasyOCR**|⭐⭐⭐⭐|보통|무료|
|Google Vision|⭐⭐⭐⭐⭐|빠름|유료|

---

## ⚠️  수정 & 고민 해야할 것 

### 1. 주소 파싱 수정함

팝업에 **주소 + 전화번호** 같이 오는 경우 있어서 정규식으로 분리함

### 2. rating/review_count 삭제

없는 전시가 너무 많아서 테이블에서 제거

### 3. 가격 필드 문제 (해결 필요)

**실제 가격 구조 (매우 복잡):**

| 유형    | 예시                             |
| ----- | ------------------------------ |
| 단일가   | 얼리버드 15,000원 (구분 없음)           |
| 연령 분리 | 성인 20,000 / 청소년 15,000         |
| 할인 분리 | 얼리버드 성인 13,800 / 단일권 성인 15,000 |
| 카드 할인 | 신한카드 성인 14,000 (30% 할인)        |
| 패키지   | 5매권 50,000 (1인 가격 아님)          |
| 기간권   | 3일권 41,000 / 1일권 20,000        |

**문제점:**

|실제 데이터|현재 처리|문제|
|---|---|---|
|성인 33,000 / 청소년 27,000|✅ 정상|-|
|얼리버드 12,000 (성인 없음)|adult=12000|⚠️ **할인가를 정가로**|
|입장권 15,000 (구분 없음)|adult=15000|⚠️ **통합권을 성인가로**|
|성인-아동청소년 공통 12,000|adult=12000|⚠️ **공통가를 성인가로**|

**예시 (CSV에서 확인):**

```
성률 기획전: price_adult=12000, price_youth=NULL
→ 실제로는 "성인-아동청소년 공통 12,000원" (얼리버드)
```

**통계 왜곡:**

```
평균 성인 가격: 18,347원  ← 얼리버드/공통권 포함됨!
```

### 4. dbt에서 해결할 방법 (TODO)

>"가격 구조가 얼리버드, 카드할인, 패키지 등 복잡해서 **raw 테이블에는 첫 번째 가격**을 저장하고, **dbt staging에서 price_type 컬럼**으로 정가/할인/단일가를 분류하면?

---

## 📊 현재 결과 (2026-04-07)

```
총 전시: 15개
OCR 추출: 15개
평균 성인 가격: 18,347원 (⚠️ 왜곡됨)
평균 청소년 가격: 13,160원

지역별: 서울 15개
```

---

## 📁 파일 구조

```
exhibition-data-mart/
├── crawl/
│   ├── crawler_selenum.py   # Selenium + OCR
│   ├── load_to_postgres.py  # DB 적재
│   ├── main.py
│   └── requirements.txt
├── postgres/
│   └── init.sql             # 테이블 DDL
├── dbt_exhibition/          # TODO
└── docker-compose.yml
```

---

## 🔌 포트

|서비스|전시 프로젝트|
|---|---|
|PostgreSQL|**5435**|
|Airflow|**8085**|
|Superset|**8090**|

---

## 📝 다음 단계

1. [ ] **dbt: 가격 유형 분류** (price_type 컬럼)
2. [ ] **dbt: 신뢰 가능 통계** (earlybird 제외)
3. [ ] Superset 대시보드
4. [ ] Airflow DAG
5. [ ] README 작성