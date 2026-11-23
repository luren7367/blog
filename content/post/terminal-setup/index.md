---
title: "我的终端与编辑器配置"
slug: "terminal-setup"
date: 2026-11-11
description: "用顺手了的几样东西：zsh、tmux、Neovim，以及一些提高效率的别名。"
image: "cover.png"
tags:
    - 工具
    - 效率
categories:
    - 技术
---

## Shell

换到 zsh 之后最常打的几个别名：

```bash
alias ll='ls -alh'
alias gs='git status -sb'
alias gp='git push'
```

## Neovim

用 lazy.nvim 管理插件，主要装了：

- `telescope.nvim`：模糊查找文件与内容
- `nvim-treesitter`：语法高亮
- `lualine.nvim`：状态栏

## 效率小习惯

1. 用 `tmux` 保持长任务会话
2. 用 `fzf` 补全历史命令：`Ctrl+R`
3. 把重复的操作写成脚本，而不是背快捷键
