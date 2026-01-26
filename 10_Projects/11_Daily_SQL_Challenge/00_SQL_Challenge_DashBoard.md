---
tags:
  - DashBoard
---
## Study Tracker

> 매일 한 문제씩 풀자!!!

```dataview
CALENDAR file.day 
FROM "10_Projects/11_Daily_SQL_Challenge" 
WHERE status != null
```

---
## 최근 푼 문제 

```dataview js
TABLE 
  split(file.name, "_")[0] as "푼 날짜",  
  split(file.name, "_")[1] as "주제"
FROM "10_Projects/11_Daily_SQL_Challenge"
WHERE file.name != "00_SQL_Challenge_DashBoard" 
SORT file.name ASC
LIMIT 5
```

---
## 복습 필요

```dataview js
TABLE 
  split(file.name, "_")[0] as "푼 날짜",  
  split(file.name, "_")[1] as "주제"
FROM "10_Projects/11_Daily_SQL_Challenge"
WHERE file.name != "00_SQL_Challenge_DashBoard" 
  AND !contains(status, "🟩 해결")
SORT file.name ASC
LIMIT 5
```



