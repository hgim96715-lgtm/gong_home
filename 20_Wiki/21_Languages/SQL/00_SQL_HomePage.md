##  핵심 사고력 (Data Engineer's Brain)

- [[SQL_Execution_Order]] : 작성 순서 vs 실행 순서 완벽 정리 (치트시트)
- [[SQL_Main_Table_Strategy]] : 쿼리의 주인공(FROM)을 정하는 절대 원칙
- [[01_SQL_Thinking_Roadmap]] : 🚨 쿼리 짜기 전 필독! (실행 순서 & 전략)
-  [[Query_Optimization]] : 쿼리 속도를 빠르게 하는 튜닝 원칙 

## Level 0. 밑그림 그리기 (Data Modeling)

> "건물을 짓기 전에 설계도부터 봐야죠. 테이블이 왜 이렇게 생겨먹었는지 이해하는 단계입니다."

- **[[Data_Modeling_Overview]]** : 모델링의 3단계와 관점 (개념/논리/물리)
- **[[ERD_Components]]** : 엔터티(Entity), 속성(Attribute), 관계(Relationship)의 정체
- **[[Keys_and_Identifiers]]** : PK(주민번호)와 FK(참조)의 관계 완벽 정리
- **[[Normalization_Theory]]** : 테이블을 쪼개는 기술, 정규화(1NF, 2NF, 3NF)


## Level 0.5. 집 짓기와 가구 채우기 (DDL & DML) 

> "설계도(모델링)가 나왔으니 이제 실제로 테이블을 생성하고(`CREATE`), 데이터를 넣어봅시다(`INSERT`)."

- **[[SQL_DDL_Create]]** : 테이블 생성과 삭제의 모든 것 (`CREATE`, `DROP`, `TRUNCATE`, `ALTER`)
- **[[SQL_DML_CRUD]]** : 데이터 넣고, 수정하고, 지우기 (`INSERT`, `UPDATE`, `DELETE`)
- **[[SQL_Constraints]]** : 이상한 데이터가 못 들어오게 막는 문지기 (`PRIMARY KEY`, `NOT NULL`, `UNIQUE`, `DEFAULT`)
- **[[SQL_Auto_Increment]]** : 번호표 자동 발급기 (`SERIAL`, `AUTO_INCREMENT`, `SEQUENCE`)


## 1. 기초 문법 (Basics)

> 데이터 조회와 필터링의 기본기

- [[SQL_SELECT_FROM]] : 데이터를 가져오는 가장 기본적인 구조
- [[SQL_Concepts_View]] : 뷰(View)란 무엇인가? (가상 테이블과 보안)
- [[Basic_Structure_and_Alias]] : 컬럼/로우 용어 정리와 별칭(AS) 짓기
-  [[SQL_Filtering_WHERE]] : 원하는 데이터만 쏙 골라내기 (AND, OR, IN, LIKE, IS NULL)
-  [[SQL_ORDER_BY]] : 데이터 줄 세우기
-  [[Data_Types]] : 문자, 숫자, 날짜 타입의 이해

## 2. 데이터 가공과 집계 (Aggregation)

> 엑셀의 피벗 테이블을 SQL로 구현하기

- [[Aggregation_GROUP_BY]] : 그룹별 통계 내기 (COUNT, SUM, AVG) 
- [[SQL_DISTINCT_vs_GROUP_BY]] : 중복만 제거할까(`DISTINCT`) vs 묶어서 계산할까(`GROUP BY`)?
- [[HAVING_vs_WHERE]] : 필터링 시점의 차이 (가장 많이 헷갈림!) 
- [[Numeric_Functions]] : 소수점 반올림, 버림, 절댓값 완벽 정리  (ROUND,TRUNC,FLOOR,CEIL )
- [[String_Functions]] : 문자열 자르고 붙이기 (SUBSTR, CONCAT) 
- [[SQL_Date_Functions]] : 날짜 더하고 빼기 (DATE_ADD, DATEDIFF)

## 3. 테이블 연결 (Joins) 

> 흩어진 데이터를 하나로 모으기

- [[SQL_JOIN_Concept]] : Inner, Left, Outer 조인의 벤다이어그램 이해 
- [[Union_vs_Join]] : 위아래로 합치기(UNION) vs 옆으로 붙이기(JOIN)

## 4. 고급 분석 (Advanced) 

> 현업에서 "오, 쿼리 좀 짜시네요?" 소리 듣는 구간

- [[Window_Functions]] : 순위, 누적합, 이동평균 (RANK, LEAD, LAG) 
- [[SubQuery_CTE]] : 복잡한 쿼리를 레고 블록처럼 관리하기 (WITH 절) 
- [[Pivot_Unpivot]] : 행을 열로, 열을 행으로 바꾸기
- [[SQL_CASE_WHEN]] : IF-ELSE 로직 구현하기 (CASE WHEN,`FILTER`,비율)

##  실전 연습 (Projects)

- [[00_SQL_Challenge_DashBoard|🏆 매일 SQL 챌린지 대시보드]] 
- [[Error_Log]] : 자주 만나는 에러 해결책 모음 (오답노트)

