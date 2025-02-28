---
title: <%*
  const yearStartOffset = 2; // 年度开头偏移量。对于 2025 年来说，该值为 2。
  const creationDate = new Date(tp.file.creation_date());
  const year = creationDate.getFullYear();
  const startOfYear = new Date(year, 0, 1);
  const dayOfYear = Math.floor((creationDate - startOfYear) / (24 * 60 * 60 * 1000)) + yearStartOffset;
  const weekNumber = Math.ceil(dayOfYear / 7) + 1;
  console.log(`Applying WeekReport: weekNumber = ${weekNumber}`)
  console.log(`Applying WeekReport: dayOfYear = ${dayOfYear}`)

  await tp.file.rename(`周报 - 第${weekNumber}周`);
  tR += `周报 - 第${weekNumber}周`;
%>
date created: <% tp.file.creation_date("YYYY-MM-DD HH:mm:ss") %>
tags:
  - "#周报/<%*
  tR += year;
%>/<%*
  tR += weekNumber;
%>"
status: Ongoing
---

<%* tp.hooks.on_all_templates_executed(async () => {
  tp.app.commands.executeCommandById("obsidian-linter:lint-file");
}); -%>

## What I Read This Week

## What I Wrote This Week

## 值得关注
