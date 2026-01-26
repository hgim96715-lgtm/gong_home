##  핵심 사고력 (Data Engineer's Brain)

- [[SQL_Execution_Order]] : 작성 순서 vs 실행 순서 완벽 정리 (치트시트)
- [[SQL_Main_Table_Strategy]] : 쿼리의 주인공(FROM)을 정하는 절대 원칙
- [[SQL_Thinking_Roadmap]] : 🚨 쿼리 짜기 전 필독! (실행 순서 & 전략)
-  [[Query_Optimization]] : 쿼리 속도를 빠르게 하는 튜닝 원칙 

## 1. 기초 문법 (Basics)

> 데이터 조회와 필터링의 기본기

- [[SQL_SELECT_FROM]] : 데이터를 가져오는 가장 기본적인 구조
- [[Basic_Structure_and_Alias]] : 컬럼/로우 용어 정리와 별칭(AS) 짓기
-  [[SQL_Filtering_WHERE]] : 원하는 데이터만 쏙 골라내기 (AND, OR, IN, LIKE, IS NULL)
-  [[SQL_ORDER_BY]] : 데이터 줄 세우기
-  [[Data_Types]] : 문자, 숫자, 날짜 타입의 이해

## 2. 데이터 가공과 집계 (Aggregation)

> 엑셀의 피벗 테이블을 SQL로 구현하기

- [[Aggregation_GROUP_BY]] : 그룹별 통계 내기 (COUNT, SUM, AVG) 
- [[HAVING_vs_WHERE]] : 필터링 시점의 차이 (가장 많이 헷갈림!) 
- [[String_Functions]] : 문자열 자르고 붙이기 (SUBSTR, CONCAT) 
- [[Date_Functions]] : 날짜 더하고 빼기 (DATE_ADD, DATEDIFF)

## 3. 테이블 연결 (Joins) 

> 흩어진 데이터를 하나로 모으기

- [[JOIN_Concept]] : Inner, Left, Outer 조인의 벤다이어그램 이해 
- [[Union_vs_Join]] : 위아래로 합치기(UNION) vs 옆으로 붙이기(JOIN)

## 4. 고급 분석 (Advanced) 

> 현업에서 "오, 쿼리 좀 짜시네요?" 소리 듣는 구간

- [[Window_Functions]] : 순위, 누적합, 이동평균 (RANK, LEAD, LAG) 
- [[SubQuery_CTE]] : 복잡한 쿼리를 레고 블록처럼 관리하기 (WITH 절) 
- [[Pivot_Unpivot]] : 행을 열로, 열을 행으로 바꾸기
- [[SQL_CASE_WHEN]] : IF-ELSE 로직 구현하기 (CASE WHEN)

##  실전 연습 (Projects)

- [[00_SQL_Challenge_DashBoard|🏆 매일 SQL 챌린지 대시보드]] 
- [[Error_Log]] : 자주 만나는 에러 해결책 모음 (오답노트)

