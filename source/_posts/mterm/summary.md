---
title: 用多语言实现一个安卓终端模拟器-总结
date: 2024-01-19 10:18:10
categories: [Project, Mterm]
tags: [Android, Linux, Tauri, Vue, Kotlin, Rust, C++]
home_cover: /images/mterm/home_cover.jpg
home_cover_height: 320
excerpt: Mterm 是一个用Tauri、Vue、Kotlin、Rust、C++ 等 Languages && Framework 实现的Android终端模拟器。此章节是对此系列的总结...
---

## Project Mterm 系列章节

系列章节:

1. [用多语言实现一个安卓终端模拟器-概述](https://blog.hackerfly.cn/2024/01/18/mterm/overview)
2. [用多语言实现一个安卓终端模拟器-libmterm](https://blog.hackerfly.cn/2024/01/19/mterm/libmterm)
3. [用多语言实现一个安卓终端模拟器-mterm](https://blog.hackerfly.cn/2024/01/19/mterm/mterm)
4. [用多语言实现一个安卓终端模拟器-mterm_packages](https://blog.hackerfly.cn/2024/01/19/mterm/mterm_packages)
5. [`用多语言实现一个安卓终端模拟器-总结`](https://blog.hackerfly.cn/2024/01/19/mterm/summary)

项目仓库：

- libmterm: [https://github.com/zakiaatot/libmterm](https://github.com/zakiaatot/libmterm)
- mterm: [https://github.com/zakiaatot/mterm](https://github.com/zakiaatot/mterm)
- mterm_packages: [https://github.com/zakiaatot/mterm_packages](https://github.com/zakiaatot/mterm_packages)

## 产品技术总结

在前面几个章节

1. 我们首先从 Linux 系统编程中实现一个伪终端开始入手，实现了一个封装出 创建终端、销毁终端、读终端、写终端 等接口的库-[libmterm](<(https://blog.hackerfly.cn/2024/01/19/mterm/libmterm)>)
   其中核心的内容就是实现伪终端的原理：
   ![pseudo terminal](/images/mterm/terminal_struct.png)

2. 然后我们基于 libmterm 和 tauri 一步步去实现终端的前端-[mterm](https://blog.hackerfly.cn/2024/01/19/mterm/mterm)
   其中的难点不是 UI 界面的编写和 xterm.js 的 api 调用，而是如何实现 JS 调用 C++ 编写的 libmterm，
   我们经过了许多层级 Js -> Rust -> C++ ,涉及 ffi 和动静态链接库等基本知识

3. 最后我们基于 termux-packages 一起实现终端的自定义 shell 环境-[mterm_packages](https://blog.hackerfly.cn/2024/01/19/mterm/mterm_packages)
   核心在于为什么需要一个自定义 shell，因为安卓自带的 /system/bin/sh 是一个非标准的 shell，仅提供少量功能，需要实现 apt 包管理器功能，就得手动编译 bash apt 等源码去实现一个自定义 shell 环境。主要涉及编译的方法和过程

最后构成了如下所示的**Project Mterm**
![screenshot](/images/mterm/screenshot.jpg)

## 不足与后续改进

主要的不足之处存在于两个地方：

1. mterm_packages 编译的不够彻底，在编译途中遇到很多问题我选择了直接跳过，这导致最终很多包无法安装，提示缺少依赖，
   在包管理和维护上还需要花大量的时间。

2. 由于 mterm 的前端基本通过 Webview+Vue 实现，可能会出现性能问题，对于终端的读取我也是通过 setInterval 来实现的，会存在某些 bug，后续优化的方案可能是放在 Kotlin 中的线程中实现。

## 收获与感悟

收获很多，从 Linux 系统编程到 Android 系统开发，从 C++ 到 Rust，从 Rust 到 Kotlin，从 Kotlin 到 Vue，学习了 伪终端编程、静动态链接库、ffi、编译 等等知识，使自我对组织项目工程的能力得到了提高。

感悟就是，想到了一个自己想做的想法，一定要去自己实现一遍，不能三分钟热度，要坚持到底，虽一路布满荆棘，但最终会有很多收获。
