---
tags:
  - Linux_Test
  - Dashboard
related:
  - "[[00_Linux_HomePage(기존)]]"
---

> [!abstract] **Engineer's Log**
> "Everything is a file."
> 터미널을 지배하는 자가 서버를 지배한다. 💻
> [Linux_Test](https://www.hackerrank.com/domains/shell)

---
## 나의 전적 (Statistics) 

```dataview
TABLE WITHOUT ID
  "🐧 **" + length(rows) + " 문제**" as "총 도전"
FROM "10_Projects/13_Daily_Linux_Challenge"
WHERE file.name != this.file.name
GROUP BY true
```

---
## Monthly Log 📅

```dataview
CALENDAR file.cday
FROM "10_Projects/13_Daily_Linux_Challenge"
```


---
## 학습 현황 (Status Board)

>[!example]+ **🚀 최신 업데이트 (Latest Commits)**

```dataview
TABLE WITHOUT ID
  link(file.link, split(file.name, "_")[1]) as "Challenge / Topic",
  split(file.name, "_")[0] as "Date",
  status as "Status"
FROM "10_Projects/13_Daily_Linux_Challenge"
WHERE file.name != this.file.name
SORT file.name DESC
LIMIT 5
```


>[!warning]+ ** 디버깅/복습 필요 (Bug Report)**

```dataview
TABLE WITHOUT ID
  link(file.link, split(file.name, "_")[1]) as "Topic",
  split(file.name, "_")[0] as "Date",
  status as "Current State"
FROM "10_Projects/13_Daily_Linux_Challenge"
WHERE file.name != this.file.name
  AND !contains(status, "🟩") 
  AND !contains(status, "🟪")
SORT file.name ASC
```

