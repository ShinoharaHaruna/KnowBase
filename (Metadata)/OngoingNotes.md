All **Ongoing** files:

```dataview
TABLE file.ctime AS "创建时间"
FROM ""
WHERE status = "Ongoing"
SORT file.ctime ASC
```

Those **Not Finished** Readings：

```dataview
TABLE file.ctime AS "创建时间"
FROM "Readings"
WHERE status != "Finished" AND file.name != "Readme"
SORT file.ctime ASC
```
