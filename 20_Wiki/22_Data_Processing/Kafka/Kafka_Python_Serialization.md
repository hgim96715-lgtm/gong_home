---
aliases:
  - 직렬화
  - 역직렬화
  - json.dumps
  - json.loads
  - encode
  - decode
  - Serialization
  - Deserialization
tags:
  - Kafka
  - Python
related:
  - "[[00_Kafka_HomePage]]"
  - "[[Kafka_Python_Producer]]"
  - "[[Kafka_Python_Consumer]]"
  - "[[Encoding_Decoding_Concept]]"
  - "[[Python_Dictionaries]]"
  - "[[Python_JSON]]"
---

# Kafka_Python_Serialization

## 개념 한 줄 요약

> **Kafka 는 bytes 만 알아들음.** 
> Python 객체 → bytes 로 변환하는 게 직렬화 (Serialization)
>  bytes → Python 객체로 복원하는 게 역직렬화 (Deserialization)

---

---

# 왜 필요한가

```
Kafka 는 내부적으로 0과 1(bytes) 만 저장하고 전송함

Python dict {"trn_no": "00051"} 을 그대로 send() 에 넣으면
  TypeError: expected bytes, got dict

반드시 bytes 로 변환(직렬화)해서 보내야 하고
받은 쪽에서는 다시 dict 로 복원(역직렬화)해야 함
```

---

---

# 직렬화 / 역직렬화 흐름

```
[Producer 쪽]
Python dict
    ↓  json.dumps(v, ensure_ascii=False)
JSON 문자열  '{"trn_no": "00051", "stn_nm": "서울"}'
    ↓  .encode("utf-8")
bytes  b'{"trn_no": "00051", "stn_nm": "\xec\x84\x9c\xec\x9a\xb8"}'
    ↓  Kafka 토픽으로 전송
────────────────────────────────────────────────────
[Consumer 쪽]
bytes  b'{"trn_no": "00051", "stn_nm": "..."}'
    ↓  .decode("utf-8")
JSON 문자열  '{"trn_no": "00051", "stn_nm": "서울"}'
    ↓  json.loads(...)
Python dict  {"trn_no": "00051", "stn_nm": "서울"}
```

---

---

# ① json.dumps — Python 객체 → JSON 문자열

```python
import json

data = {"trn_no": "00051", "stn_nm": "서울", "delay": None}

# 기본
print(json.dumps(data))
# {"trn_no": "00051", "stn_nm": "\uc11c\uc6b8", "delay": null}
#                               ↑ 한글이 유니코드로 깨짐

# ensure_ascii=False: 한글 그대로 유지 ⭐️
print(json.dumps(data, ensure_ascii=False))
# {"trn_no": "00051", "stn_nm": "서울", "delay": null}

# 보기 좋게 들여쓰기 (로그 출력용)
print(json.dumps(data, ensure_ascii=False, indent=2))
# {
#   "trn_no": "00051",
#   "stn_nm": "서울",
#   "delay": null
# }
```

```
Python → JSON 타입 변환:
  dict    → {}   (object)
  list    → []   (array)
  str     → ""   (string)
  int/float → 숫자
  True/False → true/false
  None    → null
```

---

---

# ② json.loads — JSON 문자열 → Python 객체

```python
import json

json_str = '{"trn_no": "00051", "stn_nm": "서울", "delay": null}'

data = json.loads(json_str)
print(type(data))           # <class 'dict'>
print(data["trn_no"])       # 00051
print(data["delay"])        # None  ← JSON null → Python None 자동 변환
```

```
JSON → Python 타입 변환:
  {}          → dict
  []          → list
  ""          → str
  숫자         → int / float
  true/false  → True/False
  null        → None
```

---

---

# ③ encode / decode — 문자열 ↔ bytes

```python
# 문자열 → bytes (인코딩)
text = "서울역"
b = text.encode("utf-8")
print(b)           # b'\xec\x84\x9c\xec\x9a\xb8\xec\x97\xad'
print(type(b))     # <class 'bytes'>

# bytes → 문자열 (디코딩)
restored = b.decode("utf-8")
print(restored)    # 서울역
print(type(restored))  # <class 'str'>
```

```
인코딩/디코딩 개념 자체가 궁금하면 → [[Encoding_Decoding_Concept]]
```

---

---

# ④ value_serializer / value_deserializer 패턴 정리

## Producer — value_serializer

```python
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers="kafka:9092",

    # dict → bytes 자동 변환
    value_serializer=lambda v: json.dumps(v, ensure_ascii=False).encode("utf-8"),
)

# 이제 dict 를 그냥 넣어도 됨 (serializer 가 자동 변환)
producer.send("train-realtime", value={"trn_no": "00051", "stn_nm": "서울"})
```

## Consumer — value_deserializer

```python
from kafka import KafkaConsumer
import json

consumer = KafkaConsumer(
    "train-realtime",
    bootstrap_servers="kafka:9092",

    # bytes → dict 자동 변환
    value_deserializer=lambda x: json.loads(x.decode("utf-8")),
)

for message in consumer:
    data = message.value   # 이미 dict 로 변환돼 있음
    print(data["trn_no"])  # 00051
```

---

---

# ⑤ 직렬화 패턴 종류

## 패턴 1 — JSON (가장 많이 씀)

```python
# 직렬화
lambda v: json.dumps(v, ensure_ascii=False).encode("utf-8")

# 역직렬화
lambda x: json.loads(x.decode("utf-8"))
```

```
장점: 사람이 읽기 쉬움. 디버깅 편함
단점: 텍스트라서 용량이 큼
용도: 일반 데이터 파이프라인, 학습/프로젝트
```

## 패턴 2 — 단순 문자열

```python
# 직렬화
lambda v: v.encode("utf-8")

# 역직렬화
lambda x: x.decode("utf-8")
```

```
장점: 간단함
단점: 구조화된 데이터는 별도 파싱 필요
용도: 단순 로그, 단일 값 전송
```

## 패턴 3 — serializer 없이 직접 변환 (수동)

```python
# serializer 설정 안 했을 때
producer = KafkaProducer(bootstrap_servers="kafka:9092")

# send 할 때 직접 bytes 로 변환해서 넣어야 함
data = {"trn_no": "00051"}
producer.send(
    "train-realtime",
    value=json.dumps(data, ensure_ascii=False).encode("utf-8")
)
```

```
serializer 를 쓰면 send() 마다 변환 코드 반복 안 해도 됨
serializer 설정을 권장하는 이유
```

---

---

# 실수 포인트

```
① ensure_ascii 기본값은 True
   json.dumps(data)                    → 한글이 \uXXXX 로 깨짐
   json.dumps(data, ensure_ascii=False) → 한글 그대로 유지 ← 이걸 써야 함

② value_serializer 설정 안 하고 dict 넣기
   producer.send("topic", value={"key": "val"})
   → TypeError: expected bytes
   → serializer 설정하거나 직접 .encode() 해서 넣어야 함

③ value_deserializer 설정 안 하고 .json() 처럼 쓰기
   data = message.value["trn_no"]
   → TypeError: byte indices must be integers
   → deserializer 설정 안 하면 message.value 는 bytes 그대로
   → json.loads(message.value.decode("utf-8")) 로 직접 변환해야 함

④ json.dumps 와 json.loads 헷갈리기
   dumps: Python → 문자열 (dump to string)
   loads: 문자열 → Python (load from string)
   → s 가 붙으면 string 대상
```

---

---

# 치트시트

|작업|코드|
|---|---|
|dict → JSON 문자열|`json.dumps(data, ensure_ascii=False)`|
|JSON 문자열 → dict|`json.loads(json_str)`|
|문자열 → bytes|`text.encode("utf-8")`|
|bytes → 문자열|`b.decode("utf-8")`|
|dict → bytes (한 번에)|`json.dumps(data, ensure_ascii=False).encode("utf-8")`|
|bytes → dict (한 번에)|`json.loads(b.decode("utf-8"))`|
|Producer serializer|`lambda v: json.dumps(v, ensure_ascii=False).encode("utf-8")`|
|Consumer deserializer|`lambda x: json.loads(x.decode("utf-8"))`|