---
title: "网站走向静态化"
date: 2023-02-12T22:24:17+08:00
description: "从 WordPress 走向 Hugo 的选择。"
tags: [web]
featured_image: "/images/posts/static-site-vs-traditional-site/index.png"
draft: false
comment : false
hidden: false
---

什么是静态网站？静态网站的优势？请查看 CloudFlare 博文： https://www.cloudflare.com/zh-cn/learning/performance/static-site-generator/

并非声明哪方才是正确选择，只是记录一下我的想法变化。

# 最初选择：WordPress

一开始选择 WordPress 的原因很简单：
- 教程易于跟随
- 编辑器小白友好
- 页面好看
- 主题/插件完善

但是用了一段时间 WordPress 后，也感受到了它的一些缺点：
- 性能不如静态页面
- 有一些奇奇怪怪的漏洞
- 部分功能用不到
- 编辑器体验并不好

# 转向Hugo

静态站点生成器有很多，选择Hugo是因为：
- 性能优秀
- 配套插件/主题/文档完善
- 使用 MarkDown 语法

同时，我搭配 Git 和 VSCode 等等，也得到了很优秀的编辑体验。

虽然以前对 Hugo 这类需要操作命令行与外置编辑器的使用模式并不感兴趣，但阅读了半个小时使用文档后也能完成上手。

接下来准备将 DN42 的页面和其他网站也转为静态站点，以获得更适合我的体验。