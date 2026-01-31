
> [!abstract] **Engineer's Log**
> "Everything is a file."
> 터미널을 지배하는 자가 서버를 지배한다. 💻

---
##  전투력 측정 📈

```dataview
TABLE WITHOUT ID
  "🐧 **" + length(rows) + " 문제**" as "Total Challenges",
  "🏆 **" + length(filter(rows, (r) => contains(r.status, "🟩"))) + " 개**" as "Solved",
  "🛠️ **" + length(filter(rows, (r) => !contains(r.status, "🟩"))) + " 개**" as "Refactoring Needed",
  "📈 **" + round((length(filter(rows, (r) => contains(r.status, "🟩"))) / length(rows)) * 100) + "%**" as "Completion Rate"
FROM "10_Projects/13_Daily_Linux_Challenge"
WHERE file.name != this.file.name
GROUP BY true
```

---
> [!info]+ 📅 Shell Scripting Streak

```dataview
CALENDAR file.day
FROM "10_Projects/13_Daily_Linux_Challenge"
WHERE status != null
```


---
## 프로젝트 현황 (Status Board)

>[!example]+ **🚀 최신 업데이트 (Latest Commits)**

최근 연습한 쉘 스크립트 5개를 확인합니다.

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
SORT file.name ASC
```

