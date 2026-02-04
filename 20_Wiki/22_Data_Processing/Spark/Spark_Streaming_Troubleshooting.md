---
aliases:
  - 좀비 데이터
  - 카프카 토픽 리셋
  - 스파크 체크포인트 삭제
  - 워터마크 오류 해결
tags:
  - Kafka
  - Docker
related:
  - "[[00_Apache_Spark_HomePage]]"
  - "[[Spark_Streaming_Outer_Join_Limits|Stream-Stream Outer Join]]"
---
## Concept Summary

**좀비 데이터(Zombie Data)** 란 테스트 도중 실수로 유입된 **'미래 시간(예: 2999년)'** 을 가진 데이터로, 
이 데이터가 남아있으면 시스템의 기준 시간(워터마크)이 미래로 고정되어 **현재의 정상 데이터가 모두 '지각생(Late Data)' 취급받아 버려지는 현상**을 유발합니다.

---
##  Why: 왜 이 현상을 알아채기 힘든가?

* **문제점:** 코드는 완벽하고, JSON 포맷도 맞고, 심지어 데이터가 들어오는 로그(Input Rate)도 찍힙니다. 하지만 **결과 테이블(Output)은 0건**입니다.
* **이유:** 스파크(Spark)나 플링크(Flink) 같은 스트림 엔진은 **"가장 늦은 이벤트 시간 - 여유 시간"** 으로 현재 시간을 정의합니다. 
* 누군가 실수로 `2050년` 데이터를 딱 한 건만 보내도, 시스템 시간은 `2050년`이 됩니다. 이후 들어오는 `2024년` 데이터는 **"26년이나 늦은 데이터"** 로 간주되어, 조용히 무시(Drop)됩니다.
* **해결책:** 미래 데이터를 머금고 있는 **Kafka 토픽을 삭제**하고, 그 미래 시점을 기억하고 있는 **Spark의 체크포인트(기억)** 를 모두 지워야 합니다.

---
## Practical Context: 어떤 상황에서 이 글을 떠올려야 하나?

이 해결책은 **코드를 아무리 고쳐도 결과가 안 나올 때**, 다음 **3가지 증상**이 보이면 즉시 적용해야 합니다.

### 🕵️‍♂️ 트러블슈팅 사고 과정 (Diagnosis Flow)

다음 질문에 **"Yes"** 라면 이 솔루션을 실행하세요.

1.  **"짝(Key)은 확실히 맞는가?"**
    * 데이터 1: `user_id: A`
    * 데이터 2: `user_id: A`
    * 눈으로 봤을 때 조인 조건은 완벽한데 결과가 안 나옴.

2.  **"포맷 에러는 없는가?"**
    * `null`로 변환되는 파싱 에러 로그가 없음.

3.  **"과거에 테스트를 심하게 했는가?"**
    * *"아까 테스트한다고 날짜를 2099년으로 넣어서 한번 돌려봤는데..."* 라는 기억이 있거나, 동료가 그런 테스트를 했을 가능성이 있을 때.

---
## Code Core Points

이 작업은 **"완전 초기화(Hard Reset)"** 과정입니다.

1.  **Kafka 토픽 삭제:** 잘못된 데이터(좀비)가 물리적으로 저장된 공간을 파괴합니다.
2.  **Spark 체크포인트 삭제:** **"현재 시간은 2099년이야"** 라고 기록해둔 스파크의 메모리(상태 정보)를 뇌세척(?) 합니다. 
3. **(이거 안 지우면 토픽 지워도 소용없습니다!)**

---
##  Detailed Analysis & Action Plan

터미널(macOS)을 열고 순서대로 진행하세요.

### Step 1: Kafka 토픽 날리기 (좀비 서식지 파괴)

가장 먼저 원본 데이터를 저장하고 있는 카프카 토픽을 삭제합니다.

```bash
# [해석] docker exec(컨테이너 실행) -it kafka(카프카 컨테이너에서)
# kafka-topics.sh --delete(토픽 삭제 명령) --topic [토픽명]
# --bootstrap-server(카프카 주소)
# ---------------------------------------------------------
# impression 토픽 삭제
docker exec -it kafka /opt/kafka/bin/kafka-topics.sh --delete --topic impression --bootstrap-server kafka:9092

# click 토픽 삭제
docker exec -it kafka /opt/kafka/bin/kafka-topics.sh --delete --topic click --bootstrap-server kafka:9092
```

### Step 2: Spark 체크포인트 날리기 (좀비 기억 지우기)

토픽만 지우고 다시 실행하면, 스파크가 **"어? 나 아까 2099년까지 처리했는데?"** 하고 기억(Checkpoint)을 되살려냅니다. 
이 기억도 지워야 합니다.

```bash
# 1. Spark 마스터 컨테이너 내부로 접속 (로컬 환경인 경우 생략 가능하지만, 컨테이너 환경 추천)
docker exec -it spark-master bash

# 2. 체크포인트가 저장된 임시 폴더 삭제
# [주의] 실제 운영 환경에서는 hdfs:// 경로일 수 있습니다. 여기선 로컬/학습용 경로인 /tmp 기준입니다.
# rm -rf (강제 삭제) /tmp/temporary* (체크포인트가 보통 저장되는 경로 패턴)
rm -rf /tmp/temporary*

# 3. 확인 사살 (아무것도 안 나와야 정상)
ls -l /tmp/temporary*

# 4. 컨테이너 밖으로 탈출
exit
```

### Step3 : 다시 토픽 재생성후, 다시 하면 성공 


---
## Common Beginner Misconceptions

- **"Kafka 토픽만 지우면 되겠지?"** ❌
    
    - 절대 아닙니다. Spark(또는 Flink)는 장애 복구를 위해 마지막 상태(State)를 로컬 디스크나 HDFS에 저장해둡니다.
    - 이걸 지우지 않으면 앱을 재시작하자마자 다시 미래 시점으로 돌아갑니다. **반드시 둘 다 지워야 합니다.**
        
- **"Inner Join인데 왜 안 나와?"**
    
    - 배치(Batch) 처리는 시간이 무한대라 기다려주지만, 스트림(Stream)은 **'워터마크'** 라는 마감 시간이 있습니다. 
    - 마감 시간이 지나면 조인 대상이 있어도 데이터를 버립니다.