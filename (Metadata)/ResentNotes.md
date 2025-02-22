## 最近一周的笔记

```dataview
TABLE WITHOUT ID file.link AS "文件", file.mtime AS "时间"
FROM "Learnguage" OR "Readings" OR "Week" OR "Wiki"
WHERE file.mtime >= date(today) - dur(7 days)
SORT file.mtime DESC
```

## 最近一月的笔记

```dataview
TABLE WITHOUT ID file.link AS "文件", file.mtime AS "时间"
FROM "Learnguage" OR "Readings" OR "Week" OR "Wiki"
WHERE file.mtime >= date(today) - dur(30 days)
SORT file.mtime DESC
```
