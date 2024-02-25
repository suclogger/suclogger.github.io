---
title: 配置Erlang环境
tags:
  - erlang
comments: true
toc: true
categories:
  - 编程
date: 2016-10-21 21:24:15
---

<!-- abstract -->
<!-- 开始正文 -->
本文环境：macOS Sierra
### 安装 Erlang OTP
通过homebrew安装erlang：
```
brew install erlang
```
成功页面：

![](/image/2016-11-07-15-41-12.jpg)
验证安装：
输入`erl`命令会返回：
![](/image/2016-11-07-15-42-09.jpg)

###  安装Rebar
[Rebar](http://erlang.org/doc/getting_started/seq_prog.html#id60113)可以帮助我们编译和调试erlang程序，可以通过以下命令进行安装：
```
git clone git://github.com/rebar/rebar.git
$ cd rebar
$ ./bootstrap
Recompile: src/getopt
...
Recompile: src/rebar_utils
==> rebar (compile)
$ mv ./rebar /usr/local/bin/
```
成功页面如图：

![](/image/2016-11-07-15-45-25.jpg)
### 在idea中配置erlang SDK
在插件仓库中搜索[Erlang](https://plugins.jetbrains.com/plugin/7083)进行 安装：
![](/image/2016-11-07-15-48-21.jpg)
安装成功后在添加SDK列表中就会出现erlang选项：
![](/image/2016-11-07-15-47-19.jpg)
### 在idea中配置Rebar
在`Configure | Preferences | Other Settings → Erlang External Tools`中填写路径：
```
/usr/local/bin/rebar
```

![](/image/2016-11-07-15-52-55.png)

