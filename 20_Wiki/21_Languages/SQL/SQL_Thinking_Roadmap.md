---
aliases:
  - SQL 사고 흐름도
  - 쿼리 작성 로드맵
  - SQL 실행 순서
tags:
  - SQL_Guide
related: []
---
## 개념 한 줄 요약

쿼리를 작성할 때 막막하지 않도록, **'어떤 순서로 생각하고 결정해야 하는지'** 를 데이터베이스 엔진의 실행 순서에 맞춰 시각화한 의사결정 지도입니다.

## 왜 필요한가 (Why)

**문제 상황:**
초보자는 보통 `SELECT`부터 적기 시작합니다. 
"음.. 이름이랑 매출액을 뽑아야지(SELECT)" -> "어느 테이블에서?(FROM)" -> "조건은?(WHERE)"
하지만 데이터베이스는 **정반대**로 일합니다. "데이터를 가져오고(FROM) -> 거르고(WHERE) -> 묶고(GROUP BY) -> 마지막에 보여줍니다(SELECT)".
이 생각의 순서가 꼬이면 **"Alias(별칭)를 찾을 수 없습니다"** 에러를 만나거나, 엉뚱한 집계 결과를 얻게 됩니다.

**해결책:**
이 로드맵은 **DB가 데이터를 처리하는 순서(Step 0~5)** 대로 사고하도록 강제합니다. 
특히 **"지금 서브쿼리가 필요한가?", "JOIN 하면 데이터가 늘어나지 않는가?"** 같은 실무적인 체크포인트를 미리 거치게 하여 삽질을 줄여줍니다.

## 실무 맥락에서의 사용 이유

실무에서 1,000줄짜리 복잡한 쿼리를 짤 때, 시니어 엔지니어도 머릿속으로 이 지도를 그립니다.
1.  **뻥튀기 방지:** `1:N` 조인을 할 때 무지성으로 조인하면 데이터가 수백만 배로 불어납니다. 로드맵의 `Step 1`에서 이를 경고해줍니다.
2.  **윈도우 함수 vs 그룹바이:** "행을 줄일 것인가(GROUP BY) 유지할 것인가(Window)"는 데이터 분석의 가장 핵심적인 갈림길입니다. `Step 3`가 이 결정을 도와줍니다.

## 🗺️ SQL 사고 로드맵 (Flowchart)

```mermaid
flowchart TD
    %% 시작점을 첫 번째 질문으로 바로 연결
    Start((🚀 쿼리 설계 시작)) --> Q_Complex

    %% 0. 설계 단계
    subgraph Step0_Architect ["Step 0: 큰 그림 그리기 (서브쿼리 판단)"]
        direction TB
        Q_Complex{"이미 계산된 결과(집계/순위)를<br>조건으로 걸거나<br>다시 조인해야 하는가?"}
        
        Q_Complex -- Yes: 먼저 계산 필요 --> Op_CTE["WITH 절(CTE) 또는<br>FROM 절 서브쿼리 먼저 작성<br>(Wrapper 구조)"]
        
        %% 여기서 화살표를 다음 단계의 '첫 노드'로 직접 연결
    end

    %% 1. 데이터 준비
    subgraph Step1_Data ["Step 1: 데이터 판 깔기 & 조인 전략"]
        direction TB
        Q_Sources{"데이터 소스가<br>여러 개인가?"}
        
        %% Step 0에서 Step 1의 시작점(Q_Sources)으로 연결
        Q_Complex -- No: 바로 시작 가능 --> Q_Sources
        Op_CTE --> Q_Sources

        Q_Sources -- No --> Op_From["FROM 테이블"]
        Q_Sources -- Yes --> Q_Fanout{"1:N 조인으로<br>행 수가 의도치 않게<br>늘어날 위험이 있는가?"}
        
        Q_Fanout -- No: 안전함 --> Op_Join["Standard JOIN"]
        Q_Fanout -- Yes: 위험(데이터 뻥튀기) --> Op_PreAgg["먼저 집계(GROUP BY) 후<br>조인 수행 (Pre-Aggregation)"]
        
        Op_From --> Q_Where
        Op_Join --> Q_Where
        Op_PreAgg --> Q_Where
    end

    %% 2. 필터링
    subgraph Step2_Filter ["Step 2: 재료 손질 (WHERE)"]
        direction TB
        Q_Where{"원본 데이터에서<br>미리 걸러낼 조건이 있나?<br>(날짜, 상태코드 등)"}
        Q_Where -- Yes --> Op_Where["WHERE 절 작성"]
        Q_Where -- No --> Q_Granularity
        Op_Where --> Q_Granularity
    end

    %% 3. 핵심 분기
    subgraph Step3_Core ["Step 3: 행(Row)의 운명 결정 (핵심 로직)"]
        direction TB
        Q_Granularity{"🎯 결과 집합의<br>행 수를 줄일(압축할)<br>것인가?"}

        %% === 왼쪽 트랙: GROUP BY ===
        Q_Granularity -- Yes: 그룹별 1행 --> Op_Group["GROUP BY"]
        Op_Group --> Q_AggType{"집계 목적은?"}
        Q_AggType -- 건수 --> Op_Count["COUNT"]
        Q_AggType -- 합계/평균 --> Op_SumAvg["SUM / AVG"]
        Q_AggType -- 중복제거 --> Op_Distinct["COUNT DISTINCT"]
        
        %% === 오른쪽 트랙: WINDOW ===
        Q_Granularity -- No: 행 유지 --> Q_WindowNeed{"다른 행 참조 필요?<br>(순위, 누적합, 이동평균)"}
        
        %% 바로 연결 (Step 5의 시작점 Q_Select로)
        Q_WindowNeed -- No --> Q_Select
        
        Q_WindowNeed -- Yes --> Q_Partition{"누구끼리 경쟁/합산?"}
        
        Q_Partition -- "그룹(부서)별로" --> Op_Win_Part["OVER (PARTITION BY ...)"]
        Q_Partition -- "전체 통틀어서" --> Op_Win_All["OVER ( )"]
        
        Op_Win_Part --> Q_WinOrder{"순서 중요? (랭킹/누적)"}
        Op_Win_All --> Q_WinOrder
        Q_WinOrder -- Yes --> Op_Win_Ord["ORDER BY 추가"]
        
        %% 윈도우 결과도 Step 5로 이동
        Q_WinOrder -- No --> Q_Select
        Op_Win_Ord --> Q_Select
    end

    %% 4. 후처리
    subgraph Step4_PostFilter ["Step 4: 결과 솎아내기 (HAVING)"]
        direction TB
        Op_Count --> Q_Having
        Op_SumAvg --> Q_Having
        Op_Distinct --> Q_Having
        
        Q_Having{"집계된 결과값으로<br>필터링 해야 하나?<br>(예: 매출 100이상)"}
        Q_Having -- Yes --> Op_Having["HAVING"]
        
        %% HAVING 결과도 Step 5로 이동
        Q_Having -- No --> Q_Select
        Op_Having --> Q_Select
    end

    %% 5. 마무리
    subgraph Step5_Final ["Step 5: 마무리 데코레이션"]
        direction TB
        Q_Select{"NULL 처리 필요?"}
        Q_Select -- Yes --> Op_Null["COALESCE / IFNULL"]
        Q_Select -- No --> Op_SelCol["SELECT 컬럼 확정"]
        Op_Null --> Op_SelCol
        
        Op_SelCol --> Q_Order{"최종 정렬 필요?"}
        Q_Order -- Yes --> Op_FinalOrder["ORDER BY"]
        Q_Order -- No --> Q_Limit
        Op_FinalOrder --> Q_Limit
        
        Q_Limit{"출력 개수 제한?"}
        Q_Limit -- Yes --> Op_Limit["LIMIT"]
        Q_Limit -- No --> End((🏁 완성))
    end
    
    Op_Limit --> End

    %% 스타일링
    linkStyle default stroke:#555,stroke-width:2px
    
    classDef startend fill:#2c3e50,stroke:#34495e,color:white
    class Start,End startend

    classDef decision fill:#fff9c4,stroke:#fbc02d,stroke-width:2px
    class Q_Complex,Q_Sources,Q_Fanout,Q_Where,Q_Granularity,Q_AggType,Q_WindowNeed,Q_Partition,Q_WinOrder,Q_Having,Q_Select,Q_Order,Q_Limit decision

    classDef critical fill:#ffe0b2,stroke:#e65100,stroke-width:3px
    class Q_Complex,Q_Fanout,Q_Granularity critical

    classDef action fill:#e3f2fd,stroke:#1e88e5,stroke-width:2px
    class Op_CTE,Op_From,Op_Join,Op_PreAgg,Op_Where,Op_Group,Op_Count,Op_SumAvg,Op_Distinct,Op_Win_Part,Op_Win_All,Op_Win_Ord,Op_Having,Op_Null,Op_SelCol,Op_FinalOrder,Op_Limit action

    classDef risk fill:#ffcdd2,stroke:#c62828,stroke-width:2px
    class Op_PreAgg risk
```

---
## 초보자가 자주 착각하는 포인트

1. **WHERE vs HAVING:**
    
    - `WHERE`: 데이터를 가져오자마자(그룹핑 전) 거르는 것. (가벼움)
    - `HAVING`: 그룹핑하고 계산까지 다 끝난 뒤에 거르는 것. (무거움)
    - 가능하면 `WHERE`에서 미리 다 걸러내야 쿼리가 빠릅니다.

2. **ORDER BY 위치:**
    - 쿼리를 다 짜고 맨 마지막에 하는 것입니다. 중간에 로직에 영향을 주지 않습니다 (윈도우 함수 내부 `ORDER BY` 제외).