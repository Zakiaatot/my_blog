---
title: 用多语言实现一个安卓终端模拟器-mterm
date: 2024-01-19 10:15:30
categories: [Project, Mterm]
tags: [Android, Tauri, Uniapp, Vue, Kotlin, Rust, C++]
home_cover: /images/mterm/home_cover.jpg
home_cover_height: 320
excerpt: Mterm 是一个用Tauri、Vue、Kotlin、Rust、C++ 等 Languages && Framework 实现的Android终端模拟器。此章节介绍此项目主体客户端部分mterm...
---

## Project Mterm 系列章节

系列章节:

1. [用多语言实现一个安卓终端模拟器-概述](https://blog.hackerfly.cn/2024/01/18/mterm/overview)
2. [用多语言实现一个安卓终端模拟器-libmterm](https://blog.hackerfly.cn/2024/01/19/mterm/libmterm)
3. [`用多语言实现一个安卓终端模拟器-mterm`](https://blog.hackerfly.cn/2024/01/19/mterm/mterm)
4. [用多语言实现一个安卓终端模拟器-mterm_packages](https://blog.hackerfly.cn/2024/01/19/mterm/mterm_packages)
5. [用多语言实现一个安卓终端模拟器-总结](https://blog.hackerfly.cn/2024/01/19/mterm/summary)

项目仓库：

- libmterm: [https://github.com/zakiaatot/libmterm](https://github.com/zakiaatot/libmterm)
- `mterm`: [https://github.com/zakiaatot/mterm](https://github.com/zakiaatot/mterm)
- mterm_packages: [https://github.com/zakiaatot/mterm_packages](https://github.com/zakiaatot/mterm_packages)

## 一、客户端技术选型

### 一些失败的尝试

曾经尝试过原生 Android 开发，但从未接触过原生开发的我觉得逻辑端写的很顺手，但是 UI 端让我感到很难受，于是想能不能把自己熟悉的 UI 框架 Vue 用在 Android 上。

一开始尝试过 Uniapp，最后由于种种原因被劝退了，实现的效果也不是很好，因为 Uniapp 中的 Vue 是要经过编译转成原生代码的，很多 dom 的操作都不能用，也许有人想说 Uniapp 不是有个 render.js 可以用来操作 dom 吗？的确可以操作，但是也会造成两个 script 之间通信变得困难，这使本来就需要通过多层语言调用 libmterm 到 JavaScript 的通信变得雪上加霜，所以最终还是放弃 Uniapp。后来尝试过各种类似的框架，但因为原生插件方面不给力，没法实现 Js 到 C++的调用。

### Tauri

然后想起了我之前在跨桌面端 App 开发时使用过的 Tauri，之前它还承诺过还会出移动端，并且原生支持 Rust 这种底层语言。抱着试试看的心态，并且本来就很看好 Tauri 框架的我，去看了看官网，发现它出了 2.0 Beta 版本，支持了 Android 和 Ios。按照官方的[guide](https://beta.tauri.app/guides/)很快就跑起来了一个安卓应用。

![Tauri官方搭建Mobile端指引](/images/mterm/tauri.png)

## 二、从 C++ 到 Js

### 从 C++ 到 Rust

我喜欢先从核心功能写起，已经确定是要使用 Tauri，并且 Tauri 的逻辑后端采用 Rust，最终要实现 Js 调用 C++编写的 libmterm 目标，所以第一步就是封装一个 [libmterm_rs](https://github.com/Zakiaatot/libmterm/tree/main/rs)，实现从 C++到 Rust

libmterm 目录结构:

```shell
code01@code01-A34S:~/桌面/tauri/mterm/src-tauri/lib/libmterm_rs$ tree . -L 3
.
├── Cargo.lock
├── Cargo.toml
├── lib  # 存放libmterm编译出来的静态链接库
│   └── android
│       ├── arm64-v8a
│       ├── armeabi-v7a
│       ├── x86
│       └── x86_64
├── src
│   ├── build.rs # 库编译规则
│   ├── lib.rs # libmterm的入口
│   ├── main.rs
│   └── mterm # rust封装对libmterm静态库的操作
│       ├── ffi.rs
│       ├── mod.rs
│       └── wrapper.rs

```

build.rs：

根据不同的 target 架构确定不同的静态链接库存放目录

```rust
extern crate dunce;
use std::{env, path::PathBuf};

fn main() {
    let library_name = "mterm_static";
    let root = PathBuf::from(env::var_os("CARGO_MANIFEST_DIR").unwrap());
    let target = env::var("TARGET").unwrap();

    let lib_dir;
    if target.contains("x86_64-unknown-linux-gnu") {
        lib_dir = "lib/linux";
    } else if target.contains("armv7-linux-androideabi") {
        lib_dir = "lib/android/armeabi-v7a";
    } else if target.contains("aarch64-linux-android") {
        lib_dir = "lib/android/arm64-v8a";
    } else if target.contains("i686-linux-android") {
        lib_dir = "lib/android/x86";
    } else if target.contains("x86_64-linux-android") {
        lib_dir = "lib/android/x86_64";
    } else {
        panic!("Unsupported target: {}", target);
    }

    let library_dir = dunce::canonicalize(root.join(lib_dir)).unwrap();
    println!("cargo:rustc-link-lib=stdc++");
    println!("cargo:rustc-link-lib=static={}", library_name);
    println!(
        "cargo:rustc-link-search=native={}",
        env::join_paths(&[library_dir]).unwrap().to_str().unwrap()
    );
}
```

ffi.rs:
链接静态库接口，与 [libmterm.h](https://github.com/Zakiaatot/libmterm/blob/main/include/libmterm.h) 所暴露的接口一致

```rust
pub use core::ffi::{c_char, c_int, c_uint, c_ushort};

#[link(name = "mterm_static", kind = "static")]
extern "C" {
    pub fn CreateMterm(
        cmd: *const c_char,
        cwd: *const c_char,
        argv: *const *const c_char,
        envp: *mut *mut c_char,
        rows: c_ushort,
        cols: c_ushort,
    ) -> c_int;
    pub fn CreateMtermDefault() -> c_int;
    pub fn DestroyMterm(id: c_uint) -> c_int;
    pub fn ReadMterm(id: c_uint, buf: *mut c_char, size: usize) -> c_int;
    pub fn WriteMterm(id: c_uint, buf: *const c_char, size: usize) -> c_int;
    pub fn WaitMterm(id: c_uint) -> c_int;
    pub fn SetReadNonblockMterm(id: c_uint);
    pub fn SetWindowSizeMterm(id: c_uint, rows: c_ushort, cols: c_ushort);
    pub fn CheckRunningMterm(id: c_uint) -> bool;
}
```

对静态库的再次封装：
具体代码请见 [libmterm_rs/src/mterm](https://github.com/Zakiaatot/libmterm/tree/main/rs)

一个注意的点：
由于 libmterm 的源码是用 c++ 编写的，依赖于 libc++，而在 Android 平台默认不会自动链接 libc++，故需要在 tauri 的 build.rs 中添加如下代码，告知其链接 libc++：

```rust
fn main() {
    println!("cargo:rustc-link-lib=c++"); // added
    tauri_build::build()
}
```

### 从 Rust 到 Js

Tauri 框架已经帮忙做好了封装，在 Rust 端只要将前面封装好的函数注入即可：

```rust
mod command;

#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
    tauri::Builder::default()
        .plugin(tauri_plugin_shell::init())
        .plugin(tauri_plugin_upload::init())
        .invoke_handler(tauri::generate_handler![
            command::create_mterm_default,
            command::create_mterm,
            command::destroy_mterm,
            command::read_mterm,
            command::write_mterm,
            command::wait_mterm,
            command::set_window_size_mterm,
            command::set_read_nonblock_mterm,
            command::check_running_mterm,
        ])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

而在前端，只需要用 js 封装一些异步调用即可：

```javascript
    async createMtermDefault() {
        this.id = await invoke('create_mterm_default')
    }

    async createMterm() {
        this.id = await invoke('create_mterm')
    }

    async destroyMterm() {
        return await invoke('destroy_mterm', { id: this.id })
    }

    async writeMterm(data) {
        return await invoke('write_mterm', { id: this.id, data })
    }

    async readMterm() {
        const res = await invoke('read_mterm', { id: this.id })
        this.readMsg += res
        if (this.readMsg.length > MAX_STR_LEN)
            this.readMsg = this.readMsg.slice(this.readMsg.length - MAX_STR_LEN)
        return res
    }

    async setWindowSizeMterm(rows, cols) {
        await invoke('set_window_size_mterm', { id: this.id, rows, cols })
    }

    async setReadNonblockMterm() {
        await invoke('set_read_nonblock_mterm', { id: this.id })
    }

    async checkRunningMterm() {
        return await invoke('check_running_mterm', { id: this.id })
    }
```

Tauri 这框架肥肠简单方便好不好！但是别急，这个框架存在一些问题咱们在稍后探讨。。。

## 三、愉快的前端编程之旅并不愉快

前面咱们做了这么多的工作，皆在把所有涉及系统编程的接口暴露给 Js 使用，本以为接下来就可以愉快的用 Vue 写 Ui 了用 Js 写逻辑了。

`但是我想得简单了。`

由于 Tauri 2.0 Beta 版还只是实验性的支持移动平台，很多安卓平台的操作对应的 Api 都没有，比如想要实现多颜色主题可能需要切换状态栏颜色，这个时候就需要 Kotlin 到 Js 的接口。于是我开始在 Tauri 自动生成的 Kotlin 代码上动刀。

经过仔细分析，我发现 Tauri 内部封装了 Webview 生成对应 Kotlin 代码，而且是私有的，这导致没法把我自己写的一些 Kotlin Api 注入 Webview 给 Js 调用，而且在每次重新编译时其代码会重新覆盖生成。这个时候我想到了一个妙招：在编译生成时，检查生成的代码并删除，用自己的代码来代替，这样就实现了自定义的 Js Api。

于是 mterm 仓库里就有我这一个奇奇怪怪的脚本 auto_delete_generated.sh 了:

```shell
#!/bin/bash

DIR="./src-tauri/gen/android/app/src/main/java/com/mterm/mterm/generated"

while true; do
    new_files=$(find "$DIR" -type f -cmin -1)
    for file in $new_files; do
        timestamp=$(date +"%Y-%m-%d %H:%M:%S")
        rm "$file"
        filename=$(basename "$file")
        echo -e "\e[32m$timestamp\e[0m: \e[31m$filename\e[0m deleted."
    done

    sleep 1
done
```

记得在开时调试或编译这个项目前运行这个脚本，不然会编译报错。

这样做之后还有个好处，可以自己随意封装一些 Kotlin Api 而不经过 Rust 了。

比如：

```kotlin
 inner class JsObject {
    @JavascriptInterface
    fun closeSplash() {  // js端关闭首页splash开屏页面，实现开屏缓冲
      keep = false
    }

    @JavascriptInterface
    fun setWhiteBar() {
      runOnUiThread { setStatusBarTextColor(true) } // 设置状态栏文字为黑色
    }

    @JavascriptInterface
    fun setBlackBar() {
      runOnUiThread { setStatusBarTextColor(false) } // 设置状态栏文字为白色
    }

    @SuppressLint("InternalInsetResource", "DiscouragedApi")
    @JavascriptInterface
    fun getStatusBarHeight(): Int { // 获取状态栏高度，实现键盘弹出时，webview内容不被键盘遮挡
      var height = 0
      val resourceId =
        applicationContext.resources.getIdentifier("status_bar_height", "dimen", "android")
      if (resourceId > 0) {
        height = applicationContext.resources.getDimensionPixelSize(resourceId)
      }
      val dm = applicationContext.resources.displayMetrics;
      return (height / dm.density).toInt();
    }

    @JavascriptInterface
    fun vibrate(milliseconds: Long) { // 实现震动api
      val vibrator = getSystemService(VIBRATOR_SERVICE) as Vibrator
      if (vibrator.hasVibrator()) {
        val vibrationEffect = VibrationEffect.createOneShot(milliseconds, VibrationEffect.DEFAULT_AMPLITUDE)
        vibrator.vibrate(vibrationEffect)
      }
    }

    @JavascriptInterface
    fun getOsArch(): String { // 获取cpu架构，在之后的终端环境初始化会用到
      val arch = Build.CPU_ABI
      return when {
        arch.startsWith("arm64") || arch.startsWith("aarch64") -> "aarch64"
        arch.startsWith("arm") -> "arm"
        arch.startsWith("x86_64") -> "x86_64"
        arch.startsWith("x86") -> "i686"
        else -> "unknown"
      }
    }

    @SuppressLint("SdCardPath")
    @JavascriptInterface
    fun isInit():Boolean{ // 判断终端是否初始化
        val filePath = "/data/data/com.mterm.mterm/cache/lockfile"
        val file = File(filePath)
        return file.exists()
    }
  }
```

好了，功能部分的 Api 齐全了，可以开始对 UI 的编写了。

除了功能实现外，咱们还需要一个用来解析 Terminal 数据的 UI，好在可以复用 Web 的生态，我拿出之前用到过的[xterm.js](xtermjs.org),专门就是个封装的比较完美的 Terminal 数据解析器和 UI 展示器。

其它 UI 方面没什么好说的，遇到问题解决问题即可，比如需要要用一个虚拟小键盘，那就自己手写一个。

最终的效果差不多是这个样子：

![screenshot](/images/mterm/Screenshot_20240121_110811.jpg)

## 四、总结

本篇主要讲解了基于 Tauri 框架下从 JavaScript 到 C++ 的交互，实现调用 Js 接口创建、读写、关闭伪终端，以及遇到的一些需要用原生代码 Kotlin 才能解决的问题，对前端 UI 部分的编写除了 xterm.js 以外一笔带过。

接下来我们将讲解[用多语言实现一个安卓终端模拟器-mterm_packages](https://blog.hackerfly.cn/2024/01/19/mterm/mterm_packages),涉及终端环境初始化，以及如何在安卓上集成例如 debian 的 apt 包管理器，和实现下载运行包。。。
