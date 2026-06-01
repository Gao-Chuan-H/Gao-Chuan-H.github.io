+++
title = "在 Terminator 中复制 Claude (TUI) 输出的问题排查记录"
date = "2026-05-31"
description = "记录在 Ubuntu/Terminator 下复制 Claude TUI 输出时遇到的 OSC 52 与鼠标捕获问题，以及最终的 Shift + 拖动解决方案。"
slug = "terminator-claude-tui-copy"
tags = ["Terminator", "Claude", "TUI", "终端", "Ubuntu",]
+++

## 环境

- 系统：Ubuntu 26.04
- 终端：Terminator（基于 VTE 内核）
- 场景：在终端里运行 Claude（TUI 程序），需要把界面里的文字复制到系统剪贴板

## 问题表现

1. Claude 提示 `sent chars via OSC 52`，但系统剪贴板里什么都没有。
2. 普通鼠标拖动无法选中文字，右键菜单里的"复制"是灰色的。
3. `Ctrl+Shift+C` 复制出来是空的。

## 原因分析

这个问题有两层原因，分别对应上面的两种现象。

### OSC 52 被终端静默丢弃

`OSC 52` 是一段终端转义序列（形如 `\033]52;c;<base64>\007`），程序通过它请求终端把内容写入剪贴板。

程序打印的 `sent chars via OSC 52` 只表示它发出了这段序列，不代表终端接受了。Ubuntu 默认终端（GNOME Terminal）以及 Terminator 都基于 VTE 内核，而 VTE 系终端长期不支持、也不默认启用 OSC 52 写剪贴板，收到后直接忽略。所以序列被静默丢弃，剪贴板自然为空。

这是终端的限制，不是 Claude 的问题。

### 鼠标捕获导致"无选区"

Claude 这类 TUI 程序会开启鼠标捕获（mouse tracking），你普通拖动鼠标的动作被程序吃掉，用于它自己的交互。终端本身没有高亮任何文字，也就没有选区。没有选区，右键"复制"就无内容可复制，只能灰着，`Ctrl+Shift+C` 也是空的。

## 解决办法

按住 Shift 再用鼠标拖动选择。Shift 会临时压制程序的鼠标捕获，把控制权交还给终端，这时才能真正高亮文本。

```
按住 Shift + 鼠标左键拖动选中  ->  松开  ->  Ctrl+Shift+C
```

判断成功的标志是文字出现反色高亮。有高亮说明选区生效、复制可用；没高亮说明 Shift 没压住，仍是程序在接管。

## 可选优化

### 开启"选中即复制"，省去按快捷键

```
右键 -> Preferences -> Profiles -> 选中你的 profile -> General 标签
-> 勾选 "Copy on selection"
```

勾上后，只要 Shift + 拖动选中，文字就自动进剪贴板，连 `Ctrl+Shift+C` 都不用按。

### 复制已经滚走的内容

如果要复制的文字滚出了屏幕，用 Terminator 自己的回滚缓冲翻回去再选：

```
Shift + PageUp / Shift + PageDown
```

## 其他备选方案（本次未采用）

仅作记录，适用于 Shift 选中仍失败，或希望彻底绕过终端的场景。

换支持 OSC 52 的终端：kitty、foot、WezTerm、Ghostty、Alacritty 均原生支持 OSC 52（含 SSH 远程场景）。

或者直接走管道进剪贴板。先确认会话类型：

```bash
echo $XDG_SESSION_TYPE
```

```bash
# Wayland（26.04 默认）
sudo apt install wl-clipboard
some_command | wl-copy

# X11
sudo apt install xclip
some_command | xclip -selection clipboard
```

## 结论

| 现象 | 根因 | 采用的解法 |
|------|------|-----------|
| OSC 52 复制无效 | VTE 内核不支持 OSC 52 | 改用 Shift + 选中手动复制 |
| 无法选中、复制项灰色 | TUI 程序的鼠标捕获 | 按住 Shift 拖动夺回控制权 |

最终方案：按住 Shift + 鼠标左键拖动选中 -> 松开 -> Ctrl+Shift+C
