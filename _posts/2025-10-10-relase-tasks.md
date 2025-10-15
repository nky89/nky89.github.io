---
title: "获取OM的人任务列表，方便写release notes"
date: 2025-10-10 14:30:00 -0000
tags: OM tasks "release notes" snippet
---

在写release notes的时候，有时候需要获取OM的task列表，每次都要手动的逐一写，通过工具实验，可以直接使用是 `chrome console` 来获取，直接粘贴即可

<!--more-->

代码如下，每次直接复制粘贴即可。

```js
console.log('主要功能点：\n' +
  Array.from(document.querySelectorAll('.ant-table-tbody .f-db.f-toe a'))
    .map(a =>
      `- [${a.innerText.trim()
        .replace(/【sdk-[^】]+】/g, '')
        .trim()}](${a.href.startsWith('http')
          ? a.href
          : location.origin + a.getAttribute('href')})`
    )
    .join('\n')
);

```
