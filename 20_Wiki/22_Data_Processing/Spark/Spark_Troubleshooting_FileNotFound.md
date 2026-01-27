---
aliases:
  - Py4JJavaError
  - FileNotFound
  - 스파크경로에러
  - Worker경로
tags:
  - Spark
  - Troubleshooting
  - Error
  - Docker
related:
  - "[[Spark_File_IO_Basic]]"
  - "[[Docker_Volumes]]"
---
## 에러 상황

Jupyter(Driver)에서 파일이 있는 것을 확인하고 `sc.textFile`을 돌렸는데 에러가 발생함.

```python
# 코드
sc.textFile("file:///workspace/data/file.txt").take(5)

# 에러
Py4JJavaError: An error occurred while calling ...
Caused by: java.io.FileNotFoundException: File file:/workspace/data/file.txt does not exist
```

---
## 원인: "너랑 나랑 주소가 달라" 

스파크는 **분산 시스템**입니다.

- **Driver (나):** 코드를 읽고 명령을 내림.
- **Worker (일꾼):** 실제로 파일을 읽고 계산함.

명령(`file:///...`)을 내리면 **Worker**는 **자신의 컴퓨터(컨테이너)** 에서 그 경로를 찾습니다. 
내 컴퓨터(Driver)에 파일이 있어봤자, **Worker 컴퓨터에 똑같은 경로로 파일이 없으면** 에러가 납니다.

---
## 해결책: 경로 동기화 (Sync) 

`docker-compose.yml`에서 **Driver(Jupyter)** 와 **Worker**의 `volumes` 경로를 **토씨 하나 안 틀리고 똑같이** 맞춰야 합니다.

- `spark-worker`와 `jupyter` 두 곳의 `volumes`를 똑같이 맞춰주는 것이 포인트입니다.

```YAML
services:
  # ... (Master 설정은 그대로) ...

  spark-worker:
    # ...
    volumes:
      # [수정] Worker도 이제 '/workspace/data' 경로를 씁니다.
      - ./data:/workspace/data 

  jupyter:
    # ...
    volumes:
      - ./notebooks:/workspace      # 노트북 파일 저장소
      # [수정] Jupyter도 데이터를 '/workspace/data'에서 봅니다.
      - ./data:/workspace/data
```


### 변경 사항 적용 (재시작)

설정 파일을 바꿨으니 컨테이너를 다시 띄워야 적용됩니다. 터미널에 다음 명령어를 입력하세요.

```bash
docker-compose up -d
```

- `Recreating spark-worker ... done`
- `Recreating spark-jupyter ... done`
- 이런 메시지가 뜨면 성공적으로 경로가 수정된 것입니다.