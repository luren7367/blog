---
title: "用 Hugo 与 Stack 搭起这个博客"
slug: "hello-hugo"
date: 2026-11-02
description: "记一下这个博客从零到上线用到的工具和配置，给以后的自己留个备忘。"
image: "cover.png"
tags:
    - Hugo
    - 建站
categories:
    - 技术
---

## 为什么选 Hugo

静态站点生成器里，Hugo 的构建速度几乎是最快的：几百篇文章也是毫秒级。配合 Stack 主题，开箱就有暗色模式、目录、阅读时长、标签和搜索。

## 目录结构

```text
luren-blog/
├── config/_default/    # 站点配置
├── content/            # 文章与页面
│   ├── post/           # 文章
│   └── page/           # 归档、关于、搜索
├── layouts/            # 覆盖主题的局部模板
└── themes/             # Stack 主题（git submodule）
```

## 一条命令构建

```bash
hugo --minify
```

生成的 `public/` 直接推送到 GitHub Pages 的分支即可，不需要任何服务器。

## 小结

博客的意义在于持续写下去，工具只是让这件事变得轻松一点。
