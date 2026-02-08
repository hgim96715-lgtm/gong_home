---
tags:
  - DashBoard
---
c  
> [!abstract] **Current Status**
> 쿼리 마스터가 되는 그날까지! 매일매일 꾸준함이 답이다. 🔥

---
## 나의 전적 (Statistics)

```dataview
TABLE WITHOUT ID
  "🦁 **" + length(rows) + " 문제**" as "총 도전",
  "🟩 **" + length(filter(rows, (r) => contains(r.status, "🟩"))) + " 개**" as "한방 해결",
  "🟪 **" + length(filter(rows, (r) => contains(r.status, "🟪"))) + " 개**" as "복습 성공",
  "🧨 **" + length(filter(rows, (r) => !contains(r.status, "🟩") AND !contains(r.status, "🟪"))) + " 개**" as "남은 숙제"
FROM "10_Projects/11_Daily_SQL_Challenge"
WHERE file.name != this.file.name
GROUP BY true
```

---
## Monthly Log 📅


```dataview
CALENDAR file.day FROM "10_Projects/11_Daily_SQL_Challenge" WHERE status != null
```
---
## 학습 현황 (Status Board)

>[!example]+ **🚀 최신 업데이트 (Latest Commits)**

최근 건드린 코드 5개를 확인합니다.

```dataview
TABLE WITHOUT ID
  link(file.link, split(file.name, "_")[1]) as "Project / Topic",
  split(file.name, "_")[0] as "Date",
  status as "Status"
FROM "10_Projects/11_Daily_SQL_Challenge"
WHERE file.name != this.file.name
SORT file.name DESC
LIMIT 5
```


>[!failure]+ **🔥 긴급! 복습이 필요해 (Review List)**

>아직 완벽하게 해결하지 못한(`🟩 해결` 아님) 문제들입니다

```dataview
TABLE WITHOUT ID
  link(file.link, split(file.name, "_")[1]) as "주제",
  split(file.name, "_")[0] as "날짜",
  status as "현재 상태"
FROM "10_Projects/11_Daily_SQL_Challenge"
WHERE file.name != this.file.name
  AND !contains(status, "🟩") 
  AND !contains(status, "🟪")  
SORT file.name ASC
```

---


