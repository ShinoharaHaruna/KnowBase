---
title: <% tp.file.title %>
date created: <% tp.file.creation_date("YYYY-MM-DD HH:mm:ss") %>
tags: 
status: 
---

<%* tp.hooks.on_all_templates_executed(async () => {
  const creationDate = new Date(tp.file.creation_date());
  const startOfYear = new Date(creationDate.getFullYear(), 0, 1);
  const dayOfYear = Math.floor((creationDate - startOfYear) / (24 * 60 * 60 * 1000)) + 1;
  const weekNumber = Math.ceil(dayOfYear / 7);

  await tp.file.rename(`周报 - 第${weekNumber}周`);
  tp.app.commands.executeCommandById("obsidian-linter:lint-file");
}); -%>

# <% tp.file.title %>

## 概述

简要描述笔记的内容和目的。

## 详细内容

详细记录相关信息和想法。

## 相关链接

- [[相关笔记1]]
- [[相关笔记2]]

## 参考资料

- 来源 1
- 来源 2

## 待办事项

- [ ] 任务 1
- [ ] 任务 2
