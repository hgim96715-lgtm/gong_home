## Flink Job 실행 흐름도 (Mermaid)

클라이언트가 코드를 제출(Submit)하고, 실제 태스크(Task)가 실행되기까지의 **내부 동작 과정**입니다.

```mermaid
graph TD
    %% 스타일 정의
    classDef client fill:#f9f,stroke:#333,stroke-width:2px,color:black;
    classDef jm fill:#ffcc00,stroke:#333,stroke-width:2px,color:black;
    classDef tm fill:#66ccff,stroke:#333,stroke-width:2px,color:black;
    classDef ext fill:#ddd,stroke:#333,stroke-width:2px,color:black;

    %% 1. 클라이언트 영역
    subgraph Client_Zone ["💻 Client (클라이언트)"]
        Code["Flink Code<br>(코드 작성)"] -->|"Compile<br>(컴파일)"| Graph["Dataflow Graph<br>(논리 그래프)"]
        Graph -->|"Submit Job<br>(잡 제출)"| Dispatcher
    end

    %% 2. JobManager 영역
    subgraph JobManager_Zone ["👑 JobManager (마스터 노드)"]
        Dispatcher["Dispatcher<br>(접수처)"]
        JobMaster["JobMaster<br>(작업 관리자)"]
        RM["Resource Manager<br>(자원 관리자)"]
        
        Dispatcher -->|"Spin up<br>(JM 생성)"| JobMaster
        JobMaster -->|"Request Slots<br>(슬롯 요청)"| RM
    end

    %% 3. 외부 리소스 관리자 (K8s/YARN)
    subgraph Cluster_Zone ["☁️ Cluster (인프라/K8s)"]
        Orchestrator["Orchestrator<br>(K8s / YARN)"]
    end
    
    RM -.->|"Request Container<br>(컨테이너 요청)"| Orchestrator
    Orchestrator -.->|"Start Pod<br>(TM 실행)"| TM1
    Orchestrator -.->|"Start Pod<br>(TM 실행)"| TM2

    %% 4. TaskManager 영역
    subgraph TaskManager_Zone ["👷 TaskManager (워커 노드)"]
        TM1["TaskManager 1"]
        TM2["TaskManager 2"]
        
        subgraph Slots1 [Slots]
            S1["Slot 1"]
            S2["Slot 2"]
        end
        
        subgraph Slots2 [Slots]
            S3["Slot 3"]
            S4["Slot 4"]
        end

        TM1 --- Slots1
        TM2 --- Slots2
    end

    %% 연결 관계 (흐름)
    TM1 -->|"Register<br>(등록 신고)"| RM
    TM2 -->|"Register<br>(등록 신고)"| RM
    
    RM -->|"Offer Slots<br>(슬롯 제공)"| JobMaster
    JobMaster -->|"Deploy Tasks<br>(작업 배포)"| S1
    JobMaster -->|"Deploy Tasks<br>(작업 배포)"| S2
    JobMaster -->|"Deploy Tasks<br>(작업 배포)"| S3
    
    %% 데이터 교환
    S1 <==>|"Data Exchange<br>(데이터 교환)"| S3

    %% 클래스 적용
    class Code,Graph client;
    class Dispatcher,JobMaster,RM jm;
    class TM1,TM2,S1,S2,S3,S4 tm;
    class Orchestrator ext;
```

---
### 1️⃣ 제출 (Submission)

- **Client:** 작성한 자바/파이썬 코드를 컴파일하여 논리적인 `Dataflow Graph`를 만듭니다.
- **Dispatcher:** 클라이언트의 요청을 받아주는 대문(REST API)입니다. 잡(Job)을 접수하면, 이 잡을 전담할 **`JobMaster`** (JM)를 생성합니다.

### 2️⃣ 스케줄링 & 자원 요청 (Scheduling)

- **JobMaster:** 실행 계획(JobGraph)을 물리적인 태스크로 변환하고, 이를 실행하는 데 필요한 자원(Slot)이 얼마나 필요한지 계산합니다.
- **Resource Manager:** JobMaster가 "슬롯 좀 줘"라고 요청하면, 현재 유휴 자원이 있는지 확인합니다. 없다면 K8s나 YARN에게 "컨테이너 더 띄워줘"라고 요청합니다.

### 3️⃣ 할당 & 배포 (Allocation & Deploy)

- **TaskManager(TM):** 새로 뜨거나 기존에 있던 워커들이 Resource Manager에게 "나 준비됐어(Register)"라고 신고합니다.
- **JobMaster(TM):** Resource Manager로부터 슬롯을 할당받으면, TaskManager의 **슬롯(Slot)** 에 실제 작업(Task)을 배포(Deploy)합니다.

### 4️⃣ 실행 (Execution)

- **Task Slot:** 할당받은 태스크(예: `Source`, `Map`, `Window`)를 실행합니다.
- **Data Exchange:** 서로 다른 TaskManager에 있는 슬롯끼리 네트워크를 통해 데이터를 주고받으며 스트림을 처리합니다.

---

## **코드 작성 순서(Logic Flow)** 와 **데이터 흐름(Data Flow)**

```mermaid
graph TD
    %% 스타일 정의 (색상 테마)
    classDef env fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    classDef kafka fill:#fff3e0,stroke:#e65100,stroke-width:2px;
    classDef logic fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px,stroke-dasharray: 5 5;
    classDef exec fill:#ffebee,stroke:#c62828,stroke-width:4px;
    classDef stream fill:#e1bee7,stroke:#4a148c,stroke-width:2px,stroke-dasharray: 0;

    %% 1. Environment (준비)
    subgraph Step1 ["Environment Setup (환경 설정)"]
        Env["StreamExecutionEnvironment<br/>(작업장 생성)"]
        Jar["add_jars<br/>(Kafka 통역사/JAR 주입)"]
        Env --> Jar
    end

    %% 2. Source (입력 & 연결) - 여기가 변경됨! ⭐️
    subgraph Step2 [" Source Input & Connect (설정 및 연결)"]
        KafkaIn[("Kafka Topic<br/>(input-topic)")]
        SourceCode["KafkaSource.builder<br/>(소스 부품 조립)"]
        
        %% ⭐️ 추가된 핵심 노드
        StreamInit["stream = env.from_source(...)<br/>(🌊 수도꼭지 연결 / Stream 생성)"]
        
        KafkaIn -.->|"Read (구독)"| SourceCode
        SourceCode -->|"source 객체 주입"| StreamInit
    end

    %% 3. Transformation (가공)
    subgraph Step3 [" Transformation (데이터 가공)"]
        Map["Map / FlatMap<br/>(로직: lambda x...)"]
        TypeInfo["output_type=Types...<br/>(⚠️ 직렬화 타입 명시 필수!)"]
        Map --- TypeInfo
    end

    %% 4. Sink (출력)
    subgraph Step4 ["Sink Output (데이터 저장)"]
        SinkCode["KafkaSink.builder<br/>(싱크 설정: 주소, 토픽)"]
        KafkaOut[("Kafka Topic<br/>(output-topic)")]
        SinkCode -.->|"Write (전송)"| KafkaOut
    end

    %% 5. Execute (실행)
    Step5[" Execute<br/>(실행 버튼: env.execute)"]

    %% 전체 흐름 연결
    Jar --> SourceCode
    StreamInit -->|mapped_stream = stream.map| Map
    Map -->|Processed Stream| SinkCode
    SinkCode -->|"sink_to (연결)"| Step5

    %% 스타일 적용
    class Env,Jar env;
    class KafkaIn,KafkaOut,SourceCode,SinkCode kafka;
    class Map,TypeInfo logic;
    class Step5 exec;
    class StreamInit stream;
```
