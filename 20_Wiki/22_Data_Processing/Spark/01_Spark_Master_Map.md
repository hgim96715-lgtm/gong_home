




```mermaid
graph TD
    %% ==========================================
    %% 🎨 스타일 정의 (역할 구분)
    %% ==========================================
    classDef user fill:#34495e,color:#fff;
    classDef brain fill:#2980b9,color:#fff;
    classDef plan fill:#8e44ad,color:#fff;
    classDef rdd fill:#9b59b6,color:#fff;
    classDef partition fill:#e67e22,color:#fff;
    classDef scheduler fill:#d35400,color:#fff;
    classDef task fill:#f1c40f,color:#000;
    classDef exec fill:#27ae60,color:#fff;
    classDef memory fill:#16a085,color:#fff;
    classDef data fill:#7f8c8d,color:#fff;
    classDef shuffle fill:#c0392b,color:#fff;

    %% ==========================================
    %% 1. 사용자
    %% ==========================================
    Code["👤 User Code<br/>- DataFrame / SQL 작성<br/>- 아직 실행 ❌"]:::user
    Action["🔥 Action<br/>- count / show / collect<br/>- 이 시점에 실행 시작"]:::user

    %% ==========================================
    %% 2. Driver & Catalyst
    %% ==========================================
    LogPlan["📐 Logical Plan<br/>- 어떤 컬럼을 쓰는지<br/>- 어떤 조건으로 필터하는지<br/>- 아직 데이터는 모름"]:::plan

    PhyPlan["⚙️ Physical Plan<br/>- 어떤 Join 전략을 쓸지<br/>- Shuffle 발생 여부 결정<br/>- 실행 방법 확정"]:::plan

    %% ==========================================
    %% 3. RDD (핵심 개념)
    %% ==========================================
    RDD["📄 RDD (Resilient Distributed Dataset)<br/>- Spark 내부의 실제 실행 단위 개념<br/>- 데이터 전체를 논리적으로 표현<br/>- DataFrame도 내부적으로 RDD 기반"]:::rdd

    %% ==========================================
    %% 4. Partition
    %% ==========================================
    P1["🍰 Partition 1<br/>- RDD의 일부 데이터 조각<br/>- 하나의 Task가 처리"]:::partition
    P2["🍰 Partition 2<br/>- 병렬 처리 단위<br/>- Executor 메모리에 로드됨"]:::partition

    %% ==========================================
    %% 5. Scheduler
    %% ==========================================
    DAG["🗺️ DAG Scheduler<br/>- Shuffle 기준으로 Stage 분리<br/>- 실행 순서 결정"]:::scheduler

    TaskSched["📨 Task Scheduler<br/>- Partition 1 : 1 Task 생성<br/>- Executor로 Task 전달"]:::scheduler

    %% ==========================================
    %% 6. Task
    %% ==========================================
    T1["📦 Task 1<br/>- Partition 1 처리<br/>- Filter / Map / Join 실행"]:::task
    T2["📦 Task 2<br/>- Partition 2 처리<br/>- 동일한 로직 병렬 실행"]:::task

    %% ==========================================
    %% 7. Executor & Memory
    %% ==========================================
    subgraph Exec1 ["👷 Executor 1 (JVM)"]
        Mem1["🧠 Executor Memory<br/>- cache / persist 데이터 저장<br/>- 재사용 시 파일 재읽기 ❌"]:::memory
    end

    subgraph Exec2 ["👷 Executor 2 (JVM)"]
        Mem2["🧠 Executor Memory<br/>- Shuffle 결과 저장 가능<br/>- 메모리 부족 시 Disk 사용"]:::memory
    end

    %% ==========================================
    %% 8. Shuffle
    %% ==========================================
    Shuffle["🌪️ Shuffle<br/>- Partition 간 데이터 재분배<br/>- Network + Disk IO 발생<br/>- groupBy / join 시 필수"]:::shuffle

    %% ==========================================
    %% 9. Data Source
    %% ==========================================
    File["💾 Data Source (HDFS / S3 / File)<br/>- Executor가 직접 읽음<br/>- Driver는 절대 읽지 않음"]:::data

    %% ==========================================
    %% 🔗 흐름
    %% ==========================================
    Code --> Action --> LogPlan --> PhyPlan
    PhyPlan --> RDD
    RDD --> P1
    RDD --> P2

    P1 -.-> DAG
    P2 -.-> DAG
    DAG --> TaskSched

    TaskSched --> T1 --> Exec1
    TaskSched --> T2 --> Exec2

    P1 -- groupBy / join --> Shuffle
    P2 -- groupBy / join --> Shuffle

    Shuffle --> P1
    Shuffle --> P2

    T1 -.-> File
    T2 -.-> File

    T1 --> Mem1
    T2 --> Mem2

```




---
>RDD
> **Action 이후 Executor가 실제로 실행하는 단위**
> - ⭕ **Partition = RDD 조각**
> - ⭕ 장애 나면 **RDD Lineage로 재계산**


### Grand Map 완전 해석

이 지도는 **위에서 아래로** 흐릅니다.

1. **User Layer (입력):** 사용자가 코드를 짜고 Action(`count`)을 때립니다.
2. **Catalyst (두뇌):** 스파크가 "어떻게 처리할지" 머리를 굴려 **Physical Plan**을 짭니다.
3. **Data Logic (쪼개기):**
    - Physical Plan은 **RDD**라는 논리적 설계도가 됩니다.
    - 10GB짜리 RDD는 **2개의 Partition(`Part 1`, `Part 2`)** 으로 쪼개집니다. 

4. **Scheduler (매핑):**
    - **Task Scheduler**가 파티션을 보고 **"어? 조각이 2개네? 그럼 Task도 2개!"** 하고 1:1 매핑을 합니다.
        
5. **Cluster (실행):**
    - `Task 1`은 `Executor 1`로, `Task 2`는 `Executor 2`로 배달됩니다.
    - 각 Task는 저장소(HDFS)에서 **자기 몫의 데이터(Part 1, Part 2)만 쏙 빼서(Read)** 처리합니다.