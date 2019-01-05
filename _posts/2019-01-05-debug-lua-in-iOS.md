---
layout: post
title: 在iOS中调试Lua脚本
fullview: true
category: lua
tags: lua
---


最近因工作原因，需要在 iOS 上运行 Lua 脚本，并与 Native 代码交互，正所谓工欲善其事，必先利其器，学习一门新的语言，首先要把相关环境搭建好，iOS 上 Lua 环境搭建，以及 Lua 的相关介绍本文不作讨论，在 iOS 上调试 Lua 脚本相关的文档却较少，所以在此记录一下。

# 环境

- Lua 5.1x ～ Lua5.3x
- ZeroBrane Studio [下载地址](https://studio.zerobrane.com/download?not-this-time)
- [mobdebug](https://github.com/zziking/mobdebug), 从 [AliWax](https://github.com/alibaba/wax.git) 中剥离出的工具，并对Lua5.2+ 版本做了适配

# 使用

1. 使用 ZeroBrane 打开 Lua 脚本目录
   
    ![open lua](/assets/posts/7-zerobrane.jpg)

2. 开启 debugger server
    
    ZeroBrand - >Project -> Start Debugger Server
    ![start_debugger_server](/assets/posts/7-start_debugger_server.jpg)

    ZeroBrane 将在本机的 8172 端口开启 Debugger Server

3. 将 mobdebug 添加到工程中，并添加以下代码

    {% highlight Objective-C linenos %}
    self.luaState = luaL_newstate();
    // .....
    extern void luaopen_mobdebug_scripts(void* L);
    luaopen_mobdebug_scripts(self.luaState);
    luaL_dostring(self.luaState, "require('mobdebug').start('ZeroBrane debugger server 的IP地址')")
    {% endhighlight %}


现在，打开 ZeroBrane，在 相关的 Lua 脚本上设置断点，运行工程，即可在 ZeroBrane 中调试 Lua 脚本了
  

