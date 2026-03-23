---
tags:
  - Dashboard
  - Python_Test
related:
  - "[[00_Python_HomePage]]"
---


> [!abstract] **Developer's Log**
> "Life is short, You need Python." 
> 매일 한 줄이라도 코드를 짜는 습관이 나를 만든다. 💻
 
---
## 나의 전적 (Statistics)

```dataview
TABLE WITHOUT ID
  "🐍 **" + length(rows) + " 문제**" as "총 도전"
FROM "10_Projects/12_Daily_Python_Challenge"
WHERE file.name != this.file.name
GROUP BY true
```

---
## Monthly Log 📅

```dataview
CALENDAR file.cday
FROM "10_Projects/12_Daily_Python_Challenge"
```



---
## 학습 현황 (Status Board)

>[!example]+ **🚀 최신 업데이트 (Latest Commits)**

```dataview
TABLE WITHOUT ID
  link(file.link, split(file.name, "_")[1]) as "Project / Topic",
  split(file.name, "_")[0] as "Date"
FROM "10_Projects/12_Daily_Python_Challenge"
WHERE file.name != this.file.name
SORT file.name DESC
LIMIT 5
```


