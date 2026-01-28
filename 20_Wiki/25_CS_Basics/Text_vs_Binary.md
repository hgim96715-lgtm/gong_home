---
aliases:
  - Binary
  - Text
  - rb
  - wb
  - 바이너리
  - 텍스트모드
  - UnicodeDecodeError
  - Pickle
tags:
  - CS
  - FileIO
related:
  - "[[Data_Formats_Parquet]]"
  - "[[Encoding_Concept]]"
  - "[[Python_File_IO]]"
---
## 개념 한 줄 요약 

**"사람이 읽을 수 있나(Text), 기계만 읽을 수 있나(Binary)의 차이."**

* **텍스트 파일 (Text File):** 메모장으로 열었을 때 글자가 보임. (CSV, JSON, PY, TXT)
* **바이너리 파일 (Binary File):** 메모장으로 열면 깩뛟쀍(`PNG...`)하고 깨져 보임. (이미지, 영상, 엑셀, Parquet, 실행파일)

---
## Python의 두 가지 읽기 모드 (`t` vs `b`) 

파이썬의 `open()` 함수는 기본적으로 **텍스트 모드(`t`)** 입니다. 바이너리를 다루려면 **`b`** 를 명시해야 합니다.

| 모드 | 코드 예시 | 설명 (파이썬의 행동) | 대상 파일 |
| :--- | :--- | :--- | :--- |
| **텍스트 모드** | `'r'` (read) <br> `'w'` (write) | **"이거 글자지? 내가 해독(Decoding)해줄게!"** <br> 0과 1을 가져와서 `UTF-8` 같은 규칙으로 번역해서 보여줌. | `.txt`, `.csv`, `.py`, `.json` |
| **바이너리 모드** | `'rb'` (read binary) <br> `'wb'` (write binary) | **"묻지도 따지지도 않고 있는 그대로(Raw Byte) 가져올게."** <br> 번역 안 함. 그냥 0과 1 덩어리(Bytes)로 취급함. | `.png`, `.jpg`, `.xlsx`, `.parquet`, `.pkl` |

---
## 왜 `b`를 안 붙이면 에러가 나나요? 💥

이미지 파일(`image.png`)을 텍스트 모드(`'r'`)로 열었다고 가정해 봅시다.

1.  **파이썬:** "어? `'r'` 모드네? 그럼 이 파일 안에 있는 내용을 글자(UTF-8)로 번역해야지!"
2.  **이미지 파일:** (픽셀 색상 정보인 010101110... 덩어리들)
3.  **파이썬:** "어? 이 바이트는 '가'도 아니고 'A'도 아닌데? 번역 불가능해!"
4.  **결과:** `UnicodeDecodeError: 'utf-8' codec can't decode byte...` 

> **해결책:** "야 파이썬, 그거 글자 아니야. 번역하려고 하지 마!" 라고 알려주는 게 바로 **`'rb'`** 입니다.

---
##  실제 코드 비교 

### ① 텍스트 파일 복사 (CSV)

```python
# 읽기(r), 쓰기(w) -> 기본값이 텍스트(t) 모드
with open("data.csv", "r", encoding="utf-8") as f_in:
    content = f_in.read()  # content는 '문자열(String)' 타입
    
with open("data_copy.csv", "w", encoding="utf-8") as f_out:
    f_out.write(content)
```

### ② 바이너리 파일 복사 (이미지/엑셀)

```python
# b를 꼭 붙여야 함! (encoding 옵션은 쓰면 안 됨!)
with open("photo.jpg", "rb") as f_in:
    data = f_in.read()  # data는 '바이트(Bytes)' 타입
    
with open("photo_copy.jpg", "wb") as f_out:
    f_out.write(data)
```

---
## 데이터 엔지니어링에서의 바이너리 

우리가 배운 **Parquet**나 **Excel**, 그리고 머신러닝 모델을 저장하는 **Pickle**은 모두 바이너리입니다.

- **Parquet/Excel:** 보통 `pandas`나 `spark` 라이브러리가 알아서 `'rb'`로 열어서 처리해주기 때문에 우리가 직접 `open('file.parquet', 'rb')` 할 일은 적습니다.
- **Pickle (피클):** 파이썬 객체(리스트, 딕셔너리, 모델)를 파일로 저장할 때 씁니다. 이때는 반드시 `wb`, `rb`를 써야 합니다.

```python
import pickle

my_data = {"name": "James", "age": 30}

# 저장할 때 (Write Binary)
with open("data.pkl", "wb") as f:
    pickle.dump(my_data, f)

# 불러올 때 (Read Binary)
with open("data.pkl", "rb") as f:
    loaded_data = pickle.load(f)
```

>"**메모장**으로 열어서 읽을 수 있으면 **Text(`r`)**,
>외계어가 나오거나 아예 안 열리면 **Binary(`rb`)** 야.
>스파크나 판다스는 알아서 잘 하지만,
>나중에 **이미지 처리**하거나 **AI 모델 저장(`pickle`)** 할 때는 `b` 빼먹으면 무조건 에러 나니까 꼭 기억해둬!"