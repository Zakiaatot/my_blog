---
title: 用多语言实现一个安卓终端模拟器-概述
date: 2024-01-18 15:35:17
categories: [Project, Mterm]
tags: [Android, Linux, Tauri, Vue, Kotlin, Rust, C++]
home_cover: /images/mterm/home_cover.jpg
home_cover_height: 320
excerpt: Mterm 是一个用Tauri、Vue、Kotlin、Rust、C++ 等 Languages && Framework 实现的Android终端模拟器。此章节介绍项目起源、项目大框架和系列章节安排...
---

## Project Mterm 系列章节

系列章节:

1. [`用多语言实现一个安卓终端模拟器-概述`](https://blog.hackerfly.cn/2024/01/18/mterm/overview)
2. [用多语言实现一个安卓终端模拟器-libmterm](https://blog.hackerfly.cn/2024/01/19/mterm/libmterm)
3. [用多语言实现一个安卓终端模拟器-mterm](https://blog.hackerfly.cn/2024/01/19/mterm/mterm)
4. [用多语言实现一个安卓终端模拟器-mterm_packages](https://blog.hackerfly.cn/2024/01/19/mterm/mterm_packages)
5. [用多语言实现一个安卓终端模拟器-总结](https://blog.hackerfly.cn/2024/01/19/mterm/summary)

项目仓库：

- libmterm: [https://github.com/zakiaatot/libmterm](https://github.com/zakiaatot/libmterm)
- mterm: [https://github.com/zakiaatot/mterm](https://github.com/zakiaatot/mterm)
- mterm_packages: [https://github.com/zakiaatot/mterm_packages](https://github.com/zakiaatot/mterm_packages)

## 话说上回---Mterm 项目起源

话说为何会诞生此项目，还得从去年这个时候说起。那个时候喜欢折腾手里的一台 K40，在刷 ROM、刷调度等这些常规玩法之下，逐步开始对修改编译刷入内核产生兴趣。

一直想把 Android 当作电脑来使的我，曾经最多也就下了个 Termux 然后在 chroot 环境下装上一些发行版 rootfs 然后装上 xfce 桌面跑跑 vscode 啥的。有一天在酷安刷到一条帖子，大致讲的是通过手动修改安卓内核编译配置文件可以满足开启 docker 的条件。本着好奇心的我决定自己手动给我的 K40 修改编译内核试试，也正是这次尝试，让我了解了安卓内核的编译是多么漫长的过程。

在实现在 Android 上跑 docker 之后，我发现用 Termux 从安装 docker 容器到运行容器的整个过程并不是很方便，从此我对定制一个终端模拟器产生兴趣。

一开始本想 fork 一下 Termux 然后魔改，但最终因为对 java 或者是说 androidUi 写法不太感冒而放弃，我在想能否用 Vue 作为 UI，然后用其它语言作为底层复刻一个终端模拟器。

我尝试过用 uniapp 这款框架实现了这个设想，即我之前发布的 TermDo 项目，但这个产品不算完善（应该是自己对这方面的理解还不够深入和太菜导致的），连终端环境没有编译出来，只能用安卓自带的/system/bin/sh。

也就说 Mterm 是对我之前这个愿望的重新实现（可能是因为强迫症），从底层到 UI 完全重构，也完成了虚拟 rootfs 的编译和使用。

或许哪天又会重构？于是想写下这篇博客，供自己以后使用和为其他想做类似项目尝试的用户提供帮助。

## 抛砖引玉---Mterm 项目自述

### Mterm

[![Typing SVG](https://readme-typing-svg.demolab.com?font=Fira+Code&pause=1000&color=D01DF7&vCenter=true&random=false&width=435&lines=%E4%B8%80%E4%B8%AA%E7%AE%80%E5%8D%95%E3%80%81%E6%98%93%E7%94%A8%E7%9A%84+Android+%E7%BB%88%E7%AB%AF%E6%A8%A1%E6%8B%9F%E5%99%A8%E3%80%82)](https://git.io/typing-svg)

[![Tauri](https://img.shields.io/badge/-Tauri-FF599C?style=flat-square&logo=Tauri&logoColor=ffffff)](https://tauri.app/)[![Vue.js](https://img.shields.io/badge/-Vue-4FC08D?style=flat-square&logo=Vue.js&logoColor=ffffff)](https://vuejs.org/)[![Kotlin](https://img.shields.io/badge/-Kotlin-0095D5?style=flat-square&logo=kotlin&logoColor=ffffff)](https://kotlinlang.org/)[![Rust](https://img.shields.io/badge/-Rust-000000?style=flat-square&logo=rust&logoColor=ffffff)](https://www.rust-lang.org/)[![C++](https://img.shields.io/badge/-C++-00599C?style=flat-square&logo=c%2B%2B&logoColor=ffffff)](https://en.wikipedia.org/wiki/C%2B%2B)

#### 一、项目简介

Mterm 是一个基于 [Tauri 2.0 Beta](https://beta.tauri.app/) 构建的，支持 Android 平台的终端模拟器（理论上支持 Android 全架构平台，但目前只编译了 aarch64 架构的 Packages），借鉴了 [Termux](https://github.com/termux/termux-app)。

#### 二、项目架构

1.整体语言调用层次

Vue-(webview)-Kotlin-(jni)-Rust-(ffi)-C++

2.项目构成

- [mterm](https://github.com/zakiaatot/mterm): 终端模拟器前端和业务逻辑后端，用 Tauri+Vue+Rust+Kotlin 实现

- [libmterm](https://github.com/zakiaatot/libmterm): 终端模拟器核心部分，包括伪终端的核心实现部分，用 C++ 实现，Linux 系统编程，封装出来的库提供给 mterm 使用,顺便给 java/rust 等语言封装了调用接口

- [mterm_packages](https://github.com/zakiaatot/mterm_packages): 终端模拟器的运行环境包，
  包括 bootstrap 提供的 bash 和 apt 和其他众多库包（目前只编译了 aarch64 架构），自 [termux-packages](https://github.com/termux/termux-packages) fork 修改而来，核心原理为 proot

#### 三、开发原理与过程

详情请见博客
[Zephyr's Blog-用多语言实现一个安卓终端模拟器系列](https://blog.hackerfly.cn/categories/Project/Mterm/)

#### 四、产品展示

![screenshot](/images/mterm/screenshot.jpg)

#### 五、产品下载

至 Github Release 页面下载

#### 六、其他问题

有 Bug 或什么不懂的地方或者您有更好的建议，请随时提出 issue

## 概述总览---Mterm 系列安排

根据对项目的理解，我将项目划分为项目自述中所提到的 3 个部分，即 libmterm、mterm、mterm_packages。
而我对此项目的讲解，也将分为这 3 个主体部分，外加此概述和最后总结 2 个部分。亦即总共 5 个部分。
接下来是 [libmterm](https://blog.hackerfly.cn/2024/01/19/mterm/libmterm) 的详细讲解。
