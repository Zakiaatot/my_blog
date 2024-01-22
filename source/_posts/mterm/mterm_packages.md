---
title: 用多语言实现一个安卓终端模拟器-mterm_packages
date: 2024-01-19 10:15:38
categories: [Project, Mterm]
tags: [Android, Linux, Shell]
home_cover: /images/mterm/home_cover.jpg
home_cover_height: 320
excerpt: Mterm 是一个用Tauri、Vue、Kotlin、Rust、C++ 等 Languages && Framework 实现的Android终端模拟器。此章节介绍此项目的终端环境依赖包mterm_packages以及修改编译过程...
---

## Project Mterm 系列章节

系列章节:

1. [用多语言实现一个安卓终端模拟器-概述](https://blog.hackerfly.cn/2024/01/18/mterm/overview)
2. [用多语言实现一个安卓终端模拟器-libmterm](https://blog.hackerfly.cn/2024/01/19/mterm/libmterm)
3. [用多语言实现一个安卓终端模拟器-mterm](https://blog.hackerfly.cn/2024/01/19/mterm/mterm)
4. [`用多语言实现一个安卓终端模拟器-mterm_packages`](https://blog.hackerfly.cn/2024/01/19/mterm/mterm_packages)
5. [用多语言实现一个安卓终端模拟器-总结](https://blog.hackerfly.cn/2024/01/19/mterm/summary)

项目仓库：

- libmterm: [https://github.com/zakiaatot/libmterm](https://github.com/zakiaatot/libmterm)
- mterm: [https://github.com/zakiaatot/mterm](https://github.com/zakiaatot/mterm)
- `mterm_packages`: [https://github.com/zakiaatot/mterm_packages](https://github.com/zakiaatot/mterm_packages)

## 一、为什么需要一个自定义 Shell 终端环境

安卓本身给用户提供的使用形式并不像各大 Linux 发行版那样，它不带终端，而且安卓本身所提供的 **/system/bin/sh** ，仅支持少量命令，连**ls**都可能会造成**Permisson Denied**错误，那就更不用提实现一个**apt**包管理器，任意下载使用包了。

![t1](/images/mterm/t1.jpg)

我们最终想要实现的效果，就跟电脑上的 Linux 发行版提供的 Terminal 一样，可以安装，运行自己想运行的应用，那么用安卓自带的 shell 是没法做到的,因此我们需要一个自定义 shell 环境来实现这些功能。

## 二、如何实现一个自定义 Shell 终端环境

### 1.首先分析一下 [termux-packages](https://github.com/termux/termux-packages)

俗话说的好**闭门造车不可取**，我们先来看看一些已经成熟的实现方案才能更快达到我们想要的目的。

termux-packages 是什么，Termux Packages 仓库是用于构建和管理 Termux 软件包的包构建系统。该仓库包含了一系列脚本、配置文件和描述文件，用于构建和打包各种在 Termux 上可用的软件包。这些软件包可以包括命令行工具、实用程序、编程语言和库等。

它实现自定义 Shell 终端环境的核心，就是收集了常用的包源码比如 apt bash，然后再打上修复安卓平台的 patch 然后编译到安卓平台，最后安卓可以直接执行这个二进制文件，比如 bash，就是实现的自定义终端环境，如果安装了 proot 后，可以实现基于 rootfs 的更多 Linux 发行版安装，如 debian，ubuntu，alpine，而这个就是纯纯的虚拟环境了，不知道 proot 的观众老爷可以看这篇：[Termux 与 PRoot](http://www.topv1.com/termuxdoc/proot/)

termux-packages：收集了众多包
![package](/images/mterm/package.png)

那么我们可以直接 fork [termux-packages](https://github.com/termux/termux-packages)这个仓库，然后对应我们的 app 做出修改，然后将这些包编译到我们的终端环境，这样就实现了自定义终端环境，实现 termux 的生态复用。

### 2.编译 bootstrap

经过看官方文档 [For-maintainers](https://github.com/termux/termux-packages/wiki/For-maintainers#bootstraps)
知道了第一步就是先便于一个小型带 bash、apt 等基础依赖库的环境称为**bootstrap**，然后跑起来，其它的包可以选择性编译然后上传到自建的 apt 仓库，需要时下载后安装即可。

说个比较重要的修改位置：
在 fork 下来的仓库中,**./scripts/properties.sh** 中，修改**TERMUX_APP_PACKAGE**和**TERMUX_REPO_PACKAGE**为自己的 APP 包名，比如我的是 com.mterm.mterm

其它修改可见我的 commit 记录：[compare](https://github.com/termux/termux-packages/compare/master...Zakiaatot:mterm_packages:master)

![xg1](/images/mterm/xg1.png)

修改好之后，运行 **./scripts/run-docker.sh** ,启动编译容器环境，进入后运行 **./scripts/build-bootstrap.sh**,编译 bootstrap，编译过程可能会遇到一些问题，以下是我遇到的一些问题的解决办法：

1. 404
   换下载源或版本

2. symbol not found

   ```shell
   termux_step_pre_configure() {
       export LDFLAGS="$LDFLAGS -Wl,--undefined-version"
   }
   ```

3. No package bzip2 found in any of the enabled repositories. Are you trying to set up a custom repository
   `./scripts/build-bootstrap.sh` 和 `./scripts/generate-bootstrap.sh`

   里面`bzip2`替换为`libbz2`

最后不出意外就可以编译在仓库根目录得到四不同架构的 bootstrap 包：

![bootstrap](/images/mterm/bootstrap.png)

基本启动环境就包含在里面。

### 3.对 bootstrap 的处理和 CreateMterm 的初始参数

接下来我们得到了包含 bash、apt 等基础依赖的 bootstrap 包，接下来就是将这些包部署到我们的终端环境。

我们需要在 mterm 中建一个初始化页面 init_page.vue,
然后用/system/bin/sh 先起一个安卓自带的 shell 对下载下来的 bootstrap 进行部署操作，
核心代码如下：

```javascript
 async init() {
            // download bootstrap
            const arch = AndroidApi.getOsArch()
            if (arch == "unknown") {
                return this.text = "Error: Unsupported unknown architecture!"
            }
            try {
                await download(
                    `http://repo.mterm.hackerfly.cn/bootstraps/bootstrap-${arch}.zip`,
                    `/data/data/com.mterm.mterm/bootstrap-${arch}.zip`,
                    ({ progress, total }) => {
                        this.text = `Download bootstrap: ` + (progress / total * 100).toFixed(2) + `%`
                    },
                )
            }
            catch (e) {
                return this.text = "Error: " + e
            }
            // init app
            const initShell = `
            function initApp(){
                rm -rf ${this.rootDir}
                mkdir ${this.rootDir}
                mkdir ${this.rootDir + '/usr'}
                mkdir ${this.rootDir + '/home'}
                mv /data/data/com.mterm.mterm/bootstrap-${arch}.zip ${this.rootDir + '/usr'}
                cd ${this.rootDir + '/usr'}
                chmod 777 bootstrap-${arch}.zip
                unzip bootstrap-${arch}.zip
                rm ${this.rootDir + '/usr'}/bootstrap-${arch}.zip
                cd ${this.rootDir + '/usr'}/
                for line in \`cat SYMLINKS.txt\`
                do
                    echo $line
                    OLD_IFS="\$IFS"
                    IFS="←"
                    arr=(\$line)
                    IFS="\$OLD_IFS"
                    ln -s \${arr[0]} \${arr[3]}
                done
                rm -rf SYMLINKS.txt
                TMPDIR=${this.rootDir + '/usr'}/tmp
                filename=bootstrap
                rm -rf "\$TMPDIR/\$filename*"
                rm -rf "\$TMPDIR/*"
                chmod -R 0777 ${this.rootDir + '/usr/bin'}/*
                chmod -R 0777 ${this.rootDir + '/usr'}/lib/* 2>/dev/null
                chmod -R 0777 ${this.rootDir + '/usr'}/libexec/* 2>/dev/null
                touch /data/data/com.mterm.mterm/cache/lockfile
            }
            `
            setTimeout(() => {
                this.term.writeMterm(initShell + "\n")
                this.term.writeMterm("initApp\n");
                this.text = "Init app..."
            }, 100)
            let counter = 0
            let checkInit = setInterval(() => {
                if (AndroidApi.isInit()) {
                    toast.success("Mterm init success!")
                    termManager.createTerm()
                    clearInterval(checkInit)
                    this.$router.replace('/term')
                }
                if (counter > 60) {
                    this.text = "Error: Init failed,please try again!"
                    this.term.writeMterm("rm /data/data/com.mterm.mterm/cache/lockfile\n");
                    clearInterval(checkInit)
                }
                counter++
            }, 1000)
        }
    },
```

其实本质上就是对 bootstrap 做一个 unzip 操作，然后根据里面的符号表进行软链接。

我们判断是否完成初始化的条件就是，是否存在 /data/data/com.mterm.mterm/cache/lockfile 文件，如果存在说明初始化成功，否则说明初始化失败，需要重新初始化，这个文件会在初始 shell 执行完毕后生成。

接下来就是对之后的 CreateMterm 函数进行初始化环境传参，
具体的参数如下，也是根据 Termux 的源码参考而来，主要就是把 exec 入口设置为 bootstrap 的 bash：

```rust
#[tauri::command(rename_all = "snake_case")]
pub fn create_mterm() -> i32 {
    libmterm_rs::create(
        &"/data/data/com.mterm.mterm/files/usr/bin/bash".to_string(),
        &"/data/data/com.mterm.mterm/files/home".to_string(),
        &vec!["-l".into()],
        &mut vec![
            "HOME=/data/data/com.mterm.mterm/files/home".into(),
            "TERMUX_PREFIX=/data/data/com.mterm.mterm/files/usr".into(),
            "TERM=xterm-256color".into(),
            "PATH=/data/data/com.mterm.mterm/files/usr/bin".into(),
            "LD_PRELOAD=/data/data/com.mterm.mterm/files/usr/lib/libtermux-exec.so".into(),
            "SHELL=/data/data/com.mterm.mterm/files/usr/bin/bash".into(),
            "TMPDIR=/data/data/com.mterm.mterm/files/usr/tmp".into(),
        ],
        80,
        25,
    )
}
```

接下来不出意外，就可以正常使用这个自定义的 shell 终端环境了。

### 4.编译需要的软件包

光有哪些基础包肯定不够，我们还需要编译其它要用的包，供以后需要时使用。

编译的过程也很简单，比如我自己写了个 sh 脚本，在上面提到的编译容器中执行：

```shell
#!/bin/bash
get_subdirectories() {
    local directory="$1"
    local subdirectories=""

    # 遍历目录
    for entry in "$directory"/*; do
        # 检查是否为目录
        if [ -d "$entry" ]; then
            # 获取目录名称并添加到变量中
            subdirectories+="$(basename "$entry") "
        fi
    done

    # 移除末尾的空格
    subdirectories="${subdirectories% }"

    # 返回子目录列表
    echo "$subdirectories"
}

debs=$(get_subdirectories "./packages")

# archs="aarch64 arm i686 x86_64"
archs="aarch64"

for deb in $debs; do
    for arch in $archs; do
        echo "Building $deb for $arch"
        ./build-package.sh -a "$arch" "$deb"
    done
done
```

目前因为时间和设备限制，只编译了一些 aarch64 的包。

### 5.手动建立 apt 仓库

编译出来的 deb 包，需要手动建立 apt 仓库，用来存放，我参考了下面的链接：
[termux-app 修改包名](https://ljd1996.github.io/2019/08/08/termux-app%E4%BF%AE%E6%94%B9%E5%8C%85%E5%90%8D/)
最后还是完成了仓库的建立，实现了 apt 下载软件包:

![Screenshot_20240122_123820.jpg](images/mterm/Screenshot_20240122_123820.jpg)

## 三、总结

本小结旨在参考 termux-packages 实现一个自定义的 shell 环境，而不是 Android，自带的 /system/bin/sh，可以实现软件包的下载安装功能，到此为止，我们的这个迷你安卓伪终端算是完成了，其实还有很多的功能可以往上面继续添加，下一篇我将对整个 Project Mterm 进行简单总结-[用多语言实现一个安卓终端模拟器-总结](https://blog.hackerfly.cn/2024/01/19/mterm/summary)，分享自己的收获与感悟。
