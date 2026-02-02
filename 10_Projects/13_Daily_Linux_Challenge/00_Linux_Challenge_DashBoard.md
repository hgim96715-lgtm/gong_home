[^1]
> [!abstract] **Engineer's Log**
> "Everything is a file."
> 터미널을 지배하는 자가 서버를 지배한다. 💻
> [Linux_Test](https://www.hackerrank.com/domains/shell)

---
## 나의 전적 (Statistics) 

```dataview
TABLE WITHOUT ID
  "🐧 **" + length(rows) + " 문제**" as "총 도전",
  "🟩 **" + length(filter(rows, (r) => contains(r.status, "🟩"))) + " 개**" as "한방 해결",
  "🟪 **" + length(filter(rows, (r) => contains(r.status, "🟪"))) + " 개**" as "복습 성공",
  "🧨 **" + length(filter(rows, (r) => !contains(r.status, "🟩") AND !contains(r.status, "🟪"))) + " 개**" as "남은 숙제"
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
SORT file.name ASC
```

[^1]: 
