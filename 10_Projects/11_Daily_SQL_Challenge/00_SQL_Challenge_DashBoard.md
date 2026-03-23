---
tags:
  - Dashboard
  - SQL_TEST
related:
  - "[[00_SQL_HomePage]]"
---

> [!abstract] **Current Status**
> 쿼리 마스터가 되는 그날까지! 매일매일 꾸준함이 답이다. 🔥

---
## 나의 전적 (Statistics)

```dataview
TABLE WITHOUT ID
  "🦁 **" + length(rows) + " 문제**" as "총 도전"
FROM "10_Projects/11_Daily_SQL_Challenge"
WHERE file.name != this.file.name
GROUP BY true
```

---
## Monthly Log 📅

```dataview
CALENDAR file.cday
FROM "10_Projects/11_Daily_SQL_Challenge"
```

```dataview
CALENDAR file.day
FROM "10_Projects/11_Daily_SQL_Challenge"
```
---
## 학습 현황 (Status Board)

>[!example]+ **🚀 최신 업데이트 (Latest Commits)**

최근 건드린 코드 5개를 확인합니다.

```dataview
TABLE WITHOUT ID
  link(file.link, split(file.name, "_")[1]) as "Project / Topic",
  split(file.name, "_")[0] as "Date"
FROM "10_Projects/11_Daily_SQL_Challenge"
WHERE file.name != this.file.name
SORT file.name DESC
LIMIT 5
```


---


