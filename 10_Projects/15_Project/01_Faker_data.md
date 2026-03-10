---
aliases:
  - Faker
tags:
  - Project
related:
  - "[[00_Project_HomePage]]"
  - "[[Python_Library_Faker]]"
  - "[[Kafka_Python_Producer]]"
  - "[[Python_Random_Sampling]]"
  - "[[Python_UUID]]"
  - "[[Python_DateTime]]"
linked:
  - file:////Users/gong/gong_study_de/project/generator/fake_log_producer.py
---
# Faker 라이브러리를 이용해서 가짜데이터 생성후 데이터를 **Kafka** 토픽으로 전송(Producer)

## 📌 개요

> **가짜 데이터 참고 Olist**
> - 📂 **Kaggle 원본:** [Olist E-commerce Analytics](https://www.kaggle.com/code/anshumoudgil/olist-ecommerce-analytics-quasi-poisson-poly-regs/input?select=olist_geolocation_dataset.csv)
> - 📂 **GitHub 구조:** [github-olist](https://github.com/stefansphtr/Data-Analytics-Brazilian-Ecommerce?tab=readme-ov-file#-table-of-contents)

---

## 데이터 토픽 설계 (Topic Design)

|**Kafka Topic**|**데이터 소스 (Source)**|**발생 시점 (Event Context)**|**주요 필드 (Schema)**|
|---|---|---|---|
|**`user-log`**|Python Producer (Faker)|고객이 쇼핑몰에서 **주문을 완료**했을 때|`order_id`, `total_amount`, `order_status`, `timestamp`|


---
##  주요 용어 사전 (Dictionary)


### 1. 결제 유형 (`payment_type`)

- **`credit_card`**: **신용카드**. 한국에서 가장 보편적인 결제 수단 (약 50% 비중).
- **`check_card`**: **체크카드**. 통장 잔고 내에서 즉시 출금되는 카드 (약 30% 비중).
- **`bank_transfer`**: **계좌이체/무통장입금**. 현금성 결제 수단 (약 15% 비중).
- **`voucher`**: **상품권/포인트**. (컬쳐랜드, 네이버페이 포인트 등) (약 5% 비중).

### 2. 주문 상태 (`order_status`)

- **`pending`**: 결제 대기 
- **`approved`**: 결제 승인 완료
- **`shipped`**: 택배사 발송 시작
- **`delivered`**: 고객 수령 완료 (최종 성공)
- **`canceled`**: 주문 취소

---
##  구현 코드 (Producer)

## 실행방법 

### Kafka Topic 생성

- (kafka) acbc6404d4b0

```bash
 docker exec -it acbc6404d4b0 /opt/kafka/bin/kafka-topics.sh \
  --create \
  --bootstrap-server localhost:9092 \
  --topic user-log
```

### python 파일 확인 데이터 생성기 (Producer) 실행

```bash
# 필수 라이브러리 설치
pip3 install kafka-python faker

# 파일확인 
ls -R | grep fake_log_producer.py

python3 generator/fake_log_producer.py
```

### kafka 확인 데이터 전송 확인 (Consumer)

새로운 터미널 창을 열어 카프카 안에 데이터가 실제 JSON 형태로 쌓이고 있는지 실시간으로 감시합니다.
`jq`는 터미널용 JSON 프로세서로, 복잡한 JSON에 **색깔을 입혀주고(Syntax Highlighting)**, **들여쓰기(Pretty-print)** 를 자동으로 해줍니다.

```bash
docker exec -it kafka kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic user-log \
  --from-beginning | jq
```

- **한글 깨짐:** 데이터 확인 시 `\uc2a4...`처럼 보이는 건 유니코드 표현일 뿐, 실제 데이터는 정상이니 걱정 마세요.