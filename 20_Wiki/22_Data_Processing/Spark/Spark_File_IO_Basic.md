---
aliases:
  - textFile
  - 스파크파일읽기
  - RDD생성
  - HDFS경로
  - file_protocol
tags:
  - Spark
  - RDD
related:
  - "[[RDD_Concept]]"
  - "[[Spark_Installation_Local]]"
  - "[[Docker_Volumes]]"
---
## 개념 한 줄 요약

**"스파크에게 데이터가 '어디에(Where)' 있고 '어떻게(How)' 가져와야 하는지 알려주는 방법."**

* **`sc.textFile()`:** 텍스트 파일을 읽어서 **줄(Line) 단위의 문자열 RDD**로 만듭니다.
* **프로토콜(Protocol):** 데이터가 내 컴퓨터에 있는지, 하둡에 있는지, 클라우드(S3)에 있는지 구분하는 머리말
* (`file://`, `hdfs://`, `s3a://`).

---
## 경로 작성의 법칙 (Path Rules) 

초보자가 가장 많이 틀리는 것이 바로 **경로(Path)** 입니다.

### ① 로컬 파일 읽기 (`file://`)

내 컴퓨터(정확히는 **Driver/Executor가 실행되는 컨테이너**)에 있는 파일을 읽습니다.

- `{python}문법: file:// + [절대경로]`
- 주의: 슬래시가 3개입니다! `(프로토콜 // + 루트 /)`

```python
# 문법: file:// + [절대경로]
# 주의: 슬래시가 3개입니다! (프로토콜 // + 루트 /)
rdd = sc.textFile("file:///workspace/data/sample.txt")
```

### ② 하둡(HDFS) 파일 읽기 (`hdfs://`)

현업에서 가장 많이 쓰는 방식입니다.

```python
# 문법: hdfs://[Namenode주소]:[포트]/[경로]
rdd = sc.textFile("hdfs://namenode:9000/user/data/sample.txt")

# 설정이 잘 되어 있다면 프로토콜 생략 가능 (스파크가 알아서 HDFS 뒤짐)
rdd = sc.textFile("/user/data/sample.txt")
```

### ③ 상대 경로 (Relative Path)

현재 작업 디렉토리(`WORKDIR`)를 기준으로 찾습니다.

```python
# /workspace/data/sample.txt 를 의미함
rdd = sc.textFile("data/sample.txt")
```

---
## Docker Volume과 경로의 관계 

"내 컴퓨터에 파일이 있는데 왜 못 읽지?"라고 헤맬 때 확인해야 할 **지도(Map)** 입니다.

### ① 맵핑 법칙 (Mapping Rule)

`docker-compose.yml`의 `volumes` 설정이 곧 **이정표**입니다.

```yaml
volumes:
  # [내 컴퓨터의 실제 폴더] : [컨테이너 내부의 가상 폴더]
  - ./data : /workspace/data
```

- **왼쪽 (`./data`):** 내가 파일을 복사해 넣는 곳.
- **오른쪽 (`/workspace/data`):** 스파크(컨테이너)가 파일을 찾는 곳.

### ② 코드 작성 법칙

스파크 코드는 **컨테이너 내부**에서 돌아가므로, **오른쪽 경로**를 써야 합니다.

```python
# (O) 정답: 컨테이너 내부 경로 사용 + file:// 프로토콜
# file:// + /workspace/data/word.txt
sc.textFile("file:///workspace/data/word.txt")

# (X) 오답: 내 컴퓨터 경로를 쓰면 못 찾음
sc.textFile("file:///Users/me/data/word.txt")
```

### 💡 슬래시 3개의 비밀 (`file:///`)

- **`file://`**: "이건 로컬 파일이야" (프로토콜)
- **`/home/...`**: "루트(/) 디렉토리부터 시작해" (절대 경로)
- **합치면?** -> **`file:///home/...`**

---
## 실전 예제: `word.txt` 읽기 

우리가 Docker 볼륨으로 연결한 데이터를 읽어봅시다.

```python
# 1. SparkContext 가져오기 (세션 방식)
sc = spark.sparkContext

# 2. 파일 로딩 (Transformation) - 아직 안 읽음!
# [주의] 파일이 없어도 여기서 에러 안 날 수 있음 (Lazy Evaluation)
lines = sc.textFile("file:///workspace/data/word.txt")

# 3. 데이터 확인 (Action) - 이때 파일 찾으러 감!
# 파일 없으면 여기서 "Input path does not exist" 에러 발생 🚨
print(lines.first())  # 첫 번째 줄 출력
print(lines.count())  # 전체 줄 수 세기
```

---
## 자주 묻는 질문 (FAQ)

### Q. "No such file or directory" 에러가 나요!

- **원인 1:** Docker 볼륨 연결(`- ./data:/workspace/data`)이 제대로 안 된 경우.
- **원인 2:** 오타. (`workplace` vs `workspace`)
- **원인 3:** `file://` 뒤에 슬래시를 2개만 쓴 경우 (`file://workspace/...` -> 이건 상대 경로로 인식됨).

### Q. 여러 파일을 한 번에 읽을 수 있나요?

- 네! 와일드카드(`*`)를 쓰면 됩니다.

```python
# data 폴더 안의 모든 텍스트 파일 다 읽어서 하나의 RDD로 합침
rdd = sc.textFile("data/*.txt")
```

---
###  팁

"스파크에서 경로 쓸 때 **`file:///` (슬래시 3개)** 이것만 기억해도 에러의 80%는 줄일 수 있어. 
그리고 **'내 컴퓨터(Host)'의 경로가 아니라 '컨테이너(Container)' 내부의 경로**를 써야 한다는 점! 
이게 헷갈리면 터미널에서 `docker exec -it spark-jupyter ls -R /workspace` 쳐서 파일이 어디 있는지 눈으로 확인해봐."