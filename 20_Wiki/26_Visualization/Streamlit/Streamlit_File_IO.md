---
aliases:
  - File Upload
  - File Download
  - 파일 업로드
  - 다운로드
  - CSV 처리
tags:
  - Streamlit
related:
  - "[[00_Streamlit_HomePage]]"
  - "[[Encoding_Concept]]"
---
## 개념 한 줄 요약 (Concept Summary)

**"사용자의 로컬 파일을 서버로 전송하거나(Upload), 처리된 데이터를 사용자의 PC로 저장하는(Download) 양방향 통로."**

---
## 파일 경로(Path) vs 파일 객체(Object)

사용자가 버튼을 눌러 파일을 선택하면, 파이썬이 읽을 수 있는 **임시 파일 객체(BytesIO)** 를 반환합니다
파일 경로(Path)가 아니라 **'열려 있는 파일 그 자체'** 를 준다는 점을 기억해야 합니다.

| **구분** | **일반적인 파이썬 (open())**      | **Streamlit (file_uploader)**                                     |
| ------ | -------------------------- | ----------------------------------------------------------------- |
| **정체** | **"주소 (문자열)"**             | **"물건 그 자체 (객체)"**                                                |
| **비유** | "서울시 강남구..." (집 주소가 적힌 쪽지) | 배달 받아서 **이미 내 손에 들려있는** 피자 박스                                     |
| **위치** | 하드디스크 (Disk)               | 메모리 (RAM)                                                         |
| **실수** | `pd.read_csv("data.csv")`  | `pd.read_csv("C:/Users/..." )`<br><br>(이렇게 경로를 문자열로 넣으면 **에러**남!) |

>**왜 경로(Path)가 아닐까요?**
>사용자가 웹 브라우저를 통해 파일을 올리면, 그 파일은 서버의 하드디스크에 저장되는 게 아니라 **잠시 컴퓨터 메모리(RAM)에 둥둥 떠 있는 상태(BytesIO)** 가 됩니다.
>즉, "어디에 저장됐다"는 **주소(Path) 자체가 존재하지 않습니다.** 
>그냥 "메모리에 떠 있는 저 데이터를 바로 읽어와!"라고 명령해야 합니다.

❌ 절대 하면 안 되는 코드 (Bad)

```python
file = st.file_uploader("파일 선택")

if file:
    # 에러 발생! file은 문자열(주소)이 아닙니다.
    # Streamlit은 보안상 사용자의 로컬 파일 경로(C:/Users/...)를 알 수 없습니다.
    df = pd.read_csv(file.name)
```

⭕ 올바른 코드 (Good)

```python
file = st.file_uploader("파일 선택")

if file:
    # 파일 객체 자체를 함수에 넘겨줍니다.
    # 판다스(pd.read_csv)나 PIL(Image.open)은 똑똑해서 객체를 줘도 알아서 읽습니다.
    df = pd.read_csv(file)
```

---
## 핵심 문법 완전 정복 (Core Syntax)

### ① 파일 업로드 (`st.file_uploader`)

사용자가 파일을 선택하는 버튼을 만듭니다.

```python
file = st.file_uploader("라벨", type=["csv"], accept_multiple_files=False)
```

|**인자 (Parameter)**|**필수**|**설명 (Description)**|**예시**|
|---|---|---|---|
|**`label`**|✅|업로더 위에 표시될 **제목**입니다.|`"파일을 선택하세요"`|
|**`type`**|선택|허용할 **확장자** 목록입니다. (리스트 권장)|`["csv", "xlsx"]`|
|**`accept_multiple_files`**|선택|**여러 파일**을 한 번에 올릴지 여부입니다.|`True` (리스트 반환)|
|**`key`**|선택|위젯의 고유 ID입니다. (여러 개 만들 때 필요)|`"my_uploader_1"`|

**⚠️ 주의 (Return Type):**

- 파일 없음: `None`
- 파일 1개: `UploadedFile` 객체
- 파일 N개: `[UploadedFile, ...]` 리스트

### ② 파일 다운로드 (`st.download_button`)

데이터를 내 컴퓨터로 저장하는 버튼을 만듭니다.

```python
st.download_button(label="다운로드", data=my_data, file_name="result.csv", mime="text/csv")
```

|**인자 (Parameter)**|**필수**|**설명 (Description)**|**예시**|
|---|---|---|---|
|**`label`**|✅|버튼 위에 적힐 **텍스트**입니다.|`"결과 다운로드"`|
|**`data`**|✅|다운로드할 **실제 데이터**입니다. (변환 필수!)|`csv_string` 또는 `byte_data`|
|**`file_name`**|✅|저장될 파일의 **이름**입니다. (확장자 포함)|`"report.csv"`|
|**`mime`**|선택|브라우저에게 알려줄 **파일 형식**입니다.|`"text/csv"`, `"image/png"`|


---
## 파일 유형별 처리 전략 (Cheat Sheet)

파일 종류에 따라 **"읽는 법(Read)"** 과 **"내보내는 법(Write)"** 이 다릅니다. 

|**구분**|**업로드 후 처리 (Read)**|**다운로드 전 처리 (Write)**|**필요한 버퍼**|
|---|---|---|---|
|**CSV**|`pd.read_csv(file)`|`df.to_csv(buffer)`|`io.StringIO`|
|**Text**|`file.read().decode("utf-8")`|그냥 문자열 변수 사용|필요 없음|
|**Image**|`Image.open(file)`|`img.save(buffer)`|`io.BytesIO`|

---
## CSV 파일 다루기 (Data Engineering 핵심)

### 업로드 (Upload)

`st.file_uploader`를 사용합니다.
파일이 업로드되면 `UploadedFile` 객체를 반환하며, 이걸 판다스(`pd.read_csv`)에 바로 넣을 수 있습니다.

```python
# 문법
uploaded_file = st.file_uploader("라벨", type=["csv"]) # type으로 확장자 제한

# 활용
if uploaded_file:
    # 1. 판다스로 읽기 (가장 일반적)
    df = pd.read_csv(uploaded_file)
    st.dataframe(df)
```


### 다운로드 (Download)

데이터프레임을 다운로드하려면 **문자열 버퍼(`StringIO`)** 로 변환하는 과정이 필요합니다.

```python
from io import StringIO

# 1. 버퍼 생성 (메모리상의 임시 파일)
csv_buffer = StringIO()

# 2. 데이터프레임을 버퍼에 쓰기 (인덱스 제외)
df.to_csv(csv_buffer, index=False)

# 3. 다운로드 버튼 생성
st.download_button(
    label="CSV 다운로드",
    data=csv_buffer.getvalue(),  # 버퍼에 있는 문자열 값 가져오기
    file_name="data.csv",
    mime="text/csv"              # 브라우저에게 "이건 CSV야"라고 알려줌
)
```
---
## 텍스트 파일 다루기 (Simple Text)

단순한 문자열이나 로그 파일을 처리할 때 사용합니다.

### 업로드 (Upload)

업로드된 파일은 기본적으로 **바이트(Bytes)** 상태입니다. 
사람이 읽으려면 `.decode("utf-8")`을 해줘야 합니다.

```python
uploaded_file = st.file_uploader("텍스트 업로드", type=["txt"])

if uploaded_file:
    # 1. 파일을 읽어서(read), 유니코드로 변환(decode)
    content = uploaded_file.read().decode("utf-8")
    st.text(content)
```

### 다운로드 (Download)

텍스트는 별도의 버퍼 없이 문자열(String) 변수를 그대로 `data`에 넣으면 됩니다.

```python
text_content = "안녕하세요!\n이것은 다운로드 가능한 텍스트입니다."

st.download_button(
    label="텍스트 다운로드",
    data=text_content,      # 문자열 변수 그대로 사용
    file_name="log.txt",
    mime="text/plain"       # MIME 타입: 일반 텍스트
)
```

---
## 이미지 파일 다루기 (Binary Data)

이미지나 엑셀 파일은 **바이트(Binary)** 데이터이므로 처리가 조금 다릅니다.

### 업로드 (Upload)

`PIL`(Pillow) 라이브러리의 `Image.open()`을 사용해 엽니다.

```python
from PIL import Image

uploaded_file = st.file_uploader("이미지 업로드", type=["png", "jpg"])

if uploaded_file:
    # 1. 이미지 열기
    img = Image.open(uploaded_file)
    # 2. 화면에 표시 (use_container_width=True: 화면 폭에 맞춤)
    st.image(img, caption="업로드된 이미지", use_container_width=True)
```

### 다운로드 (Download)

이미지는 텍스트가 아니므로 **바이트 버퍼(`BytesIO`)** 를 사용해야 합니다.

```python
from io import BytesIO
from PIL import Image

# 1. 바이트 버퍼 생성
buf = BytesIO()

# 2. 이미지를 버퍼에 저장 (형식 지정 필수: PNG/JPEG)
img.save(buf, format="PNG")

# 3. 버퍼에서 바이트 값 꺼내기
byte_im = buf.getvalue()

st.download_button(
    label="이미지 다운로드",
    data=byte_im,            # 바이트 데이터
    file_name="image.png",
    mime="image/png"         # MIME 타입: 이미지
)
```

---
## ⚠️ 필수 체크 포인트 (Limit)

1. **용량 제한:** Streamlit은 기본적으로 **200MB**까지만 업로드를 허용합니다.
2. **설정 변경:** 더 큰 파일을 올리려면 프로젝트 폴더의 `.streamlit/config.toml` 파일에서 설정을 바꿔야 합니다.

```TOML
[server]
maxUploadSize = 500
```
