




```mermaid
graph TD
    %% ==========================================
    %% 🎨 스타일 정의 (시각적 구분)
    %% ==========================================
    classDef user fill:#34495e,stroke:#fff,stroke-width:2px,color:#fff;
    classDef brain fill:#2980b9,stroke:#fff,stroke-width:4px,color:#fff;
    classDef logic fill:#8e44ad,stroke:#fff,stroke-width:2px,color:#fff;
    classDef partition fill:#e67e22,stroke:#fff,stroke-width:2px,color:#fff;
    classDef scheduler fill:#d35400,stroke:#fff,stroke-width:2px,color:#fff;
    classDef task fill:#f1c40f,stroke:#333,stroke-width:2px,color:#000;
    classDef worker fill:#27ae60,stroke:#fff,stroke-width:4px,color:#fff;
    classDef data fill:#7f8c8d,stroke:#fff,stroke-width:2px,color:#fff;

    %% ==========================================
    %% 1. DRIVER NODE (두뇌 & 계획 & 분배)
    %% ==========================================
    subgraph Driver ["🧠 Driver Node (작전 사령부)"]
        direction TB

        %% 1-1. 사용자 요청
        subgraph UserLayer ["👤 User Request"]
            Code["Code<br/>(DataFrame/SQL)"]:::user
            Action["🔥 ACTION!<br/>(count, collect)"]:::user
        end

        %% 1-2. 두뇌 (최적화)
        subgraph Catalyst ["⚙️ Catalyst Optimizer (지능)"]
            LogPlan["Logical Plan<br/>(무엇을?)"]:::brain
            PhyPlan["Physical Plan<br/>(어떻게?)"]:::brain
        end

        %% 1-3. RDD & 파티션 로직 (여기가 합쳐진 핵심!)
        subgraph DataLogic ["📄 Data Logic (쪼개기 전략)"]
            RDD["RDD (논리적 전체)"]:::logic
            
            subgraph Partitions ["🔪 Partitioning (조각내기)"]
                P1["🍰 Part 1"]:::partition
                P2["🍰 Part 2"]:::partition
            end
        end

        %% 1-4. 스케줄링 (배송)
        subgraph SchedulerLayer ["📅 Scheduling (배차)"]
            DAG["🗺️ DAG Scheduler<br/>(Stage 분할)"]:::scheduler
            
            subgraph TaskSched ["📨 Task Scheduler (1:1 매핑)"]
                Map1["Part 1 ➔ Task 1"]:::scheduler
                Map2["Part 2 ➔ Task 2"]:::scheduler
            end
        end
    end

    %% ==========================================
    %% 2. CLUSTER / WORKERS (실행 현장)
    %% ==========================================
    subgraph Cluster ["🏗️ Cluster (실행 현장)"]
        direction TB

        %% Executor 1
        subgraph Exec1 ["👷 Executor 1"]
            T1["📦 <b>Task 1</b><br/>(Read Part1 + Calc)"]:::task
        end

        %% Executor 2
        subgraph Exec2 ["👷 Executor 2"]
            T2["📦 <b>Task 2</b><br/>(Read Part2 + Calc)"]:::task
        end
    end

    %% ==========================================
    %% 3. DATA SOURCE (저장소)
    %% ==========================================
    File[("💾 HDFS / S3 / File<br/>(10GB Data)")]:::data

    %% ==========================================
    %% 🔗 연결 흐름 (Flow)
    %% ==========================================
    
    %% 1. 코드 -> 최적화
    Code --> Action --> LogPlan --> PhyPlan
    
    %% 2. 최적화 -> RDD 생성 -> 파티션 분할
    PhyPlan -- "RDD 변환" --> RDD
    RDD -- "Split" --> P1
    RDD -- "Split" --> P2
    
    %% 3. 파티션 정보를 DAG Scheduler가 참조
    P1 -.-> DAG
    P2 -.-> DAG
    DAG --> TaskSched
    
    %% 4. Task Scheduler가 파티션을 Task로 포장 (1:1)
    P1 === Map1
    P2 === Map2
    
    %% 5. Task 배송 (Executor로)
    Map1 -- "전송" --> Exec1
    Map2 -- "전송" --> Exec2
    
    %% 6. 실제 데이터 읽기 (Task가 직접 함)
    T1 -.-> File
    T2 -.-> File
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