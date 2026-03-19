---
aliases:
  - NativeCodeLoader
  - 하둡라이브러리경고
  - Spark경고로그
  - builtin-java
  - 스파크빨간로그
tags:
  - Spark
  - Troubleshooting
related:
  - "[[Spark_Session_Deep_Dive]]"
  - "[[Spark_Installation_Local_Docker]]"
  - "[[00_Apache_Spark_HomePage]]"
---
## 로그 내용 (Log Message)

스파크(`SparkSession`)를 처음 실행하면 터미널이나 Jupyter 로그 창에 아래와 같은 빨간색 문구가 뜹니다.

```text
WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... 
using builtin-java classes where applicable
```

---
## 진단 및 분석 (Diagnosis)

### ① 결론

**"지극히 정상입니다. (에러 아님, 무시 가능)"** 

### ② 상황 설명

- 스파크는 하둡 파일 시스템(HDFS) 등을 다룰 때, 더 빠른 속도를 내기 위해 운영체제(OS)에 최적화된 **C언어 라이브러리(Native Library)** 를 찾아서 쓰려고 시도합니다.
- 하지만 우리가 설치한 Docker 컨테이너 환경(Linux/AMD64 등)에 딱 맞는 Native 파일이 미리 세팅되어 있지 않은 상황입니다.

### ③ 스파크의 대처

- **"Native 라이브러리를 못 찾았네? 괜찮아. 그냥 자바(Java)가 원래 가지고 있는 기능(builtin-java classes)으로 대신 처리할게."** 라고 안내하는 것입니다.

---
## 영향 (Impact)

- **기능:** 100% 정상 작동합니다. 기능상 아무런 차이가 없습니다.
- **성능:** 대규모 상용 클러스터에서는 미세한 입출력 속도 차이가 있을 수 있지만, **학습용이나 로컬 개발 환경에서는 차이를 전혀 느낄 수 없습니다.**

>스파크에서 **`WARN`** 으로 시작하는 건 '내가 알아서 처리했으니까 너는 신경 꺼'라는 뜻이 대부분이야. 
>이 로그는 **'스파크가 살아있다'는 신호**로 받아들이면 돼!"