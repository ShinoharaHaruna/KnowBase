# 标签

```dataviewjs
let tags = new Map();

for (let page of dv.pages()) {
    for (let tag of page.file.tags) {
        tags.set(tag, (tags.get(tag) || 0) + 1);
    }
}

let table = "| Tag | Count |\n| --- | --- |\n";

for (let [tag, count] of Array.from(tags.entries()).sort((a, b) => b[1] - a[1])) {
    table += `| ${tag} | ${count} |\n`;
}

// Display the table
dv.paragraph(table);

// Display the Markdown code block
dv.paragraph("## 表格源码")
dv.paragraph("```markdown\n" + table + "\n```");
```
