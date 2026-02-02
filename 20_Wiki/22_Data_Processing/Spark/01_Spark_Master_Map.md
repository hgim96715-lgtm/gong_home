




```mermaid
graph TD
    %% ==========================================
    %% 🎨 스타일 정의 (가독성 UP)
    %% ==========================================
    classDef user fill:#2c3e50,stroke:#fff,stroke-width:2px,color:#fff;
    classDef driver fill:#8e44ad,stroke:#fff,stroke-width:2px,color:#fff;
    classDef optimizer fill:#e74c3c,stroke:#fff,stroke-width:4px,color:#fff;
    classDef scheduler fill:#d35400,stroke:#fff,stroke-width:2px,color:#fff;
    classDef executor fill:#27ae60,stroke:#fff,stroke-width:2px,color:#fff;
    classDef memory fill:#16a085,stroke:#fff,stroke-width:2px,color:#fff;
    classDef data fill:#7f8c8d,stroke:#fff,stroke-width:2px,color:#fff;
    classDef shuffle fill:#c0392b,stroke:#f1c40f,stroke-width:4px,color:#fff;
    classDef optimization fill:#f39c12,stroke:#333,stroke-width:2px,color:#000;

    %% ==========================================
    %% 1. 사용자 영역 (Level 1)
    %% ==========================================
    subgraph User_Space ["👤 사용자 영역 (Driver Program)"]
        Code["📜 Code (Lazy Evaluation)<br/>- DataFrame / SQL<br/>- 변환(Transformation) 정의"]:::user
        
        Action["🔥 Action (Trigger)<br/>- count(), show(), save()<br/>- 실제 실행 시작!"]:::user
    end

    %% ==========================================
    %% 2. 최적화 & 계획 영역 (Level 2 & 3)
    %% ==========================================
    subgraph Driver_Brain ["🧠 스파크의 뇌 (Driver & Optimizer)"]
        
        Catalyst["⚙️ Catalyst Optimizer<br/>- 논리적 계획 -> 물리적 계획<br/>- Join 순서 결정, Filter Pushdown"]:::driver
        
        subgraph Opt_Techs ["🚀 최적화 기술 (Optimization)"]
            Hints["💡 SQL Hints<br/>(Broadcast, Merge 강제)"]:::optimization
            AQE["🤖 AQE (Adaptive Query Execution)<br/>- 실행 중 계획 수정<br/>- Skew Join 해결, Coalesce"]:::optimizer
            DPP["✂️ DPP<br/>(Dynamic Partition Pruning)<br/>- 필요 없는 파티션 미리 제거"]:::optimization
        end
        
        DAG["🗺️ DAG Scheduler<br/>- Shuffle 기준으로 Stage 나누기<br/>- Wide vs Narrow Dependency"]:::scheduler
        
        TaskSched["📨 Task Scheduler<br/>- 각 Executor에 Task 배분<br/>- Locality(데이터 위치) 고려"]:::scheduler
        
        Speculation["🐢 Speculative Execution<br/>- 낙오자(Straggler) 발견 시<br/>- 복제 Task 실행"]:::optimization
    end

    %% ==========================================
    %% 3. 실행 & 데이터 영역 (Level 0 & 2)
    %% ==========================================
    subgraph Cluster_Farm ["🏢 클러스터 (Executors & Resources)"]
        
        subgraph Worker_Node ["👷 Worker Node (일꾼)"]
            
            Exec["⚙️ Executor (JVM Process)"]:::executor
            
            subgraph Task_Types ["📦 Tasks (병렬 처리)"]
                TaskNormal["Task (Normal)<br/>- Filter, Map, Project"]:::executor
                TaskSpec["Task (Speculative)<br/>- 느린 놈 대신 투입!"]:::optimization
            end

            subgraph Memory_Zone ["🧠 Memory & Features"]
                Cache["💾 Cache / Persist<br/>- 반복 계산 방지<br/>(Storage Memory)"]:::memory
                UnifiedMem["⚡ Unified Memory<br/>- 실행(Execution) vs 저장(Storage)<br/>- 유동적 공유"]:::memory
                Accum["📊 Accumulator<br/>- 전역 카운터 (디버깅/집계)<br/>- Driver로 결과 전송"]:::optimization
            end
        end

        Shuffle["🌪️ Shuffle (Exchange)<br/>- 네트워크/디스크 I/O 발생<br/>- groupBy, Join 시 필수<br/>- 비용 비쌈!"]:::shuffle
        
        Broadcast["📡 Broadcast Variable<br/>- 작은 테이블은 모든 노드에 복제<br/>- Shuffle 제거 (Join 최적화)"]:::optimization
    end

    %% ==========================================
    %% 4. 데이터 소스 (Level 4)
    %% ==========================================
    File["💾 Data Source (Storage)<br/>- Parquet, CSV, JSON<br/>- Partitions 단위로 읽음"]:::data

    %% ==========================================
    %% 🔗 연결 흐름 (Flow)
    %% ==========================================
    
    %% 사용자 -> 드라이버
    Code --> Action
    Action --> Catalyst

    %% 최적화 기술 적용
    Hints -.-> Catalyst
    AQE -.-> Catalyst
    DPP -.-> Catalyst
    Catalyst --> DAG

    %% 스케줄링
    DAG --> TaskSched
    TaskSched --> Speculation
    TaskSched -- "Task 할당" --> Exec

    %% 실행 및 데이터 처리
    File -- "Partition Read" --> TaskNormal
    Exec --> TaskNormal
    Exec --> TaskSpec
    
    %% 메모리 및 기능
    TaskNormal --> Cache
    TaskNormal -- "통계 전송" --> Accum
    Accum -.-> Driver_Brain

    %% 셔플 및 브로드캐스트
    TaskNormal -- "Join/GroupBy" --> Shuffle
    Shuffle --> TaskNormal
    Broadcast -.-> TaskNormal

    %% 레전드 연결 (레이아웃용)
    UnifiedMem --- Cache
```


---

### 이 지도를 읽는 법 (범례)

1. **파란색 (User):** 우리가 작성하는 코드입니다. `Action`을 때리기 전까진 아무 일도 안 일어납니다 (Lazy).
    
2. **보라색/주황색 (Driver):** 스파크의 두뇌입니다.
    - **Catalyst:** 코드를 분석해서 최적의 경로를 짭니다.
    - **Scheduler:** 작업을 스테이지(Stage)와 태스크(Task)로 쪼개서 일꾼에게 던집니다.

3. **노란색/빨간색 (Optimization):** **Level 3에서 배운 핵심 기술들**입니다.
    - **Hints, AQE, DPP:** 더 똑똑한 계획을 짜도록 도와줍니다.
    - **Speculation:** 느린 태스크를 처리합니다.
    - **Broadcast:** 셔플 비용을 없앱니다.

4. **초록색 (Executor):** 실제 일을 하는 일꾼입니다
    - **Cache:** 자주 쓰는 건 메모리에 저장합니다.
    - **Accumulator:** 작업 현황(카운트)을 드라이버에게 보고합니다.

5. **빨간색 (Shuffle):** **성능의 주적!** 데이터가 네트워크를 타고 섞이는 과정입니다. (최대한 피해야 함)