---
layout:     post
title:      linux
subtitle:   linux 运维 bash快捷键
date:       2019-10-18
author:     BY xiaobao(微信：Bao1697047283)
header-img: img/post-bg-mi1.jpg
catalog: true
tags:
    - blog
    - linux
    
---

# bash快捷键

## 光标移动

```bash
快捷键	描述
Ctrl + a	移动光标到行首
Ctrl + e	移动光标到行尾
Alt + b	移动光标后退一个单词（词首）
Alt + f	移动光标前进一个单词（词首）
Ctrl + f	光标前进一个字母
Ctrl + b	光标后退一个字母
Ctrl + xx	当前位置与行首之间光标切换
```

## 剪切粘贴

快捷键	描述
Ctrl + k	删除从光标到行尾
Ctrl + u	删除从光标到行首
Ctrl + w	从光标向前删除一个单词
Alt + d	从光标向后删除一个单词
Ctrl + d	删除光标下一个字母
Ctrl + h	删除光标前一个字母
Alt + t	swap(当前单词, 上一个单词)
Ctrl + t	swap(当前字母, 上一个字母)
Ctrl + y	粘贴上一次删除的文本


## 大小写转换

```bash
快捷键	 描述
Alt + c	大写当前字母，并移动光标到单词尾
Alt + u	大写从当光标到单词尾
Alt + l	小写从当光标到单词尾
```

## 历史命令

```bash
快捷键	描述
Ctrl + r	向后搜索历史命令
Ctrl + g	退出搜索
Ctrl + p	历史中上一个命令
Ctrl + n	历史中下一个命令
Alt + .	上一个命令的最后一个单词
```

## 终端指令

```bash
快捷键	描述
Ctrl + l	清屏
Ctrl + s	停止输出（在Zsh中为向前搜索历史命令）
Ctrl + q	继续输出
Ctrl + c	终止当前命令
Ctrl + z	挂起当前命令
Ctrl + d	结束输入（产生一个EOF）
```