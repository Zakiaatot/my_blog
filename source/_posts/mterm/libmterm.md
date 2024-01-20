---
title: 用多语言实现一个安卓终端模拟器-libmterm
date: 2024-01-19 10:15:19
categories: [Project, Mterm]
tags: [Android, Linux, Kotlin, Rust, C++]
home_cover: /images/mterm/home_cover.jpg
home_cover_height: 320
excerpt: Mterm 是一个用Tauri、Vue、Kotlin、Rust、C++ 等 Languages && Framework 实现的Android终端模拟器。此章节介绍伪终端原理和其实现库libmterm...
---

## Project Mterm 系列章节

系列章节:

1. [用多语言实现一个安卓终端模拟器-概述](https://blog.hackerfly.cn/2024/01/18/mterm/overview)
2. [`用多语言实现一个安卓终端模拟器-libmterm`](https://blog.hackerfly.cn/2024/01/19/mterm/libmterm)
3. [用多语言实现一个安卓终端模拟器-mterm](https://blog.hackerfly.cn/2024/01/19/mterm/mterm)
4. [用多语言实现一个安卓终端模拟器-mterm_packages](https://blog.hackerfly.cn/2024/01/19/mterm/mterm_packages)
5. [用多语言实现一个安卓终端模拟器-总结](https://blog.hackerfly.cn/2024/01/19/mterm/summary)

项目仓库：

- `libmterm`: [https://github.com/zakiaatot/libmterm](https://github.com/zakiaatot/libmterm)
- mterm: [https://github.com/zakiaatot/mterm](https://github.com/zakiaatot/mterm)
- mterm_packages: [https://github.com/zakiaatot/mterm_packages](https://github.com/zakiaatot/mterm_packages)

## Unix 标准终端实现原理

这种终端通常称为伪终端，平常在诸如 Gnome、xfce 等桌面环境里打开的 Terminal，也是这种终端，我们最终想要实现的其实也是这种终端，只不过前端变成了安卓罢了。接下来先看看互联网对伪终端的科普介绍：

注：以下科普来自 [Linux 伪终端(pty) - sparkdev - 博客园](https://www.cnblogs.com/sparkdev/p/11605804.html)

### 伪终端

伪终端(pseudo terminal，有时也被称为 pty)是指伪终端 master 和伪终端 slave 这一对字符设备。其中的 slave 对应 /dev/pts/ 目录下的一个文件，而 master 则在内存中标识为一个文件描述符(fd)。伪终端由终端模拟器提供，终端模拟器是一个运行在用户态的应用程序。

Master 端是更接近用户显示器、键盘的一端，slave 端是在虚拟终端上运行的 CLI(Command Line Interface，命令行接口)程序。Linux 的伪终端驱动程序，会把 master 端(如键盘)写入的数据转发给 slave 端供程序输入，把程序写入 slave 端的数据转发给 master 端供(显示器驱动等)读取。请参考下面的示意图(此图来自互联网)：

![pseudo terminal](/images/mterm/terminal_struct.png)

### 伪终端的实现原理

伪终端的基本原理涉及以下几个关键组件和步骤：

1. 主设备（Master Device）：主设备是伪终端的一端，它充当终端设备的角色，通常以 /dev/ptmx 的形式存在。主设备提供了一个接口，允许应用程序打开和控制伪终端会话。

2. 从设备（Slave Device）：从设备是伪终端的另一端，它是与主设备配对的设备。从设备通常以 /dev/pts/n 的形式存在，其中 n 是从设备的编号。从设备连接到终端模拟器或其他应用程序，它将终端设备的输入和输出转发给主设备。

3. 打开主设备：应用程序通过打开 /dev/ptmx 主设备来请求创建一个新的伪终端会话。该操作返回一个文件描述符，用于与新的伪终端会话进行通信。

4. 获取从设备：通过调用 grantpt() 和 unlockpt() 函数，应用程序可以获取与主设备配对的从设备的路径。这样，应用程序可以打开从设备以进行输入和输出操作。

5. 进程通信：应用程序可以通过读取和写入与伪终端会话关联的文件描述符来进行进程间通信。从设备将终端输入（如键盘输入）转发给主设备，主设备将其转发给应用程序。应用程序的输出通过主设备发送到从设备，然后传递给终端模拟器或其他应用程序。

## 基于 Linux 的终端实现分析

有了原理，我们就尝试实现一个简易的伪终端后端，因为涉及系统调用，为了方便使用系统库函数和方便代码组织，我选择了 C++，当然你用任何其他底层语言都是可行的，Rust、C、Zig 等。

### 首先分析一下 Termux App 的实现

Termux 关于伪终端的核心实现非常简短但精悍，只封装了一个只有 214 行的单文件 C 通过 JNI 调用。该部分位于[termux-app/terminal-emulator/src/main/jni/termux.c](https://github.com/termux/termux-app/blob/master/terminal-emulator/src/main/jni/termux.c)

其核心创建伪终端代码如下：

```c
static int create_subprocess(JNIEnv* env,
        char const* cmd,
        char const* cwd,
        char* const argv[],
        char** envp,
        int* pProcessId,
        jint rows,
        jint columns)
{
    int ptm = open("/dev/ptmx", O_RDWR | O_CLOEXEC);
    if (ptm < 0) return throw_runtime_exception(env, "Cannot open /dev/ptmx");

#ifdef LACKS_PTSNAME_R
    char* devname;
#else
    char devname[64];
#endif
    if (grantpt(ptm) || unlockpt(ptm) ||
#ifdef LACKS_PTSNAME_R
            (devname = ptsname(ptm)) == NULL
#else
            ptsname_r(ptm, devname, sizeof(devname))
#endif
       ) {
        return throw_runtime_exception(env, "Cannot grantpt()/unlockpt()/ptsname_r() on /dev/ptmx");
    }

    // Enable UTF-8 mode and disable flow control to prevent Ctrl+S from locking up the display.
    struct termios tios;
    tcgetattr(ptm, &tios);
    tios.c_iflag |= IUTF8;
    tios.c_iflag &= ~(IXON | IXOFF);
    tcsetattr(ptm, TCSANOW, &tios);

    /** Set initial winsize. */
    struct winsize sz = { .ws_row = (unsigned short) rows, .ws_col = (unsigned short) columns };
    ioctl(ptm, TIOCSWINSZ, &sz);

    pid_t pid = fork();
    if (pid < 0) {
        return throw_runtime_exception(env, "Fork failed");
    } else if (pid > 0) {
        *pProcessId = (int) pid;
        return ptm;
    } else {
        // Clear signals which the Android java process may have blocked:
        sigset_t signals_to_unblock;
        sigfillset(&signals_to_unblock);
        sigprocmask(SIG_UNBLOCK, &signals_to_unblock, 0);

        close(ptm);
        setsid();

        int pts = open(devname, O_RDWR);
        if (pts < 0) exit(-1);

        dup2(pts, 0);
        dup2(pts, 1);
        dup2(pts, 2);

        DIR* self_dir = opendir("/proc/self/fd");
        if (self_dir != NULL) {
            int self_dir_fd = dirfd(self_dir);
            struct dirent* entry;
            while ((entry = readdir(self_dir)) != NULL) {
                int fd = atoi(entry->d_name);
                if (fd > 2 && fd != self_dir_fd) close(fd);
            }
            closedir(self_dir);
        }

        clearenv();
        if (envp) for (; *envp; ++envp) putenv(*envp);

        if (chdir(cwd) != 0) {
            char* error_message;
            // No need to free asprintf()-allocated memory since doing execvp() or exit() below.
            if (asprintf(&error_message, "chdir(\"%s\")", cwd) == -1) error_message = "chdir()";
            perror(error_message);
            fflush(stderr);
        }
        execvp(cmd, argv);
        // Show terminal output about failing exec() call:
        char* error_message;
        if (asprintf(&error_message, "exec(\"%s\")", cmd) == -1) error_message = "exec()";
        perror(error_message);
        _exit(1);
    }
}
```

下面是对代码的详细分析和总结：

1. 首先，在函数的开头通过 open 函数打开/dev/ptmx 设备文件，获取一个主终端的文件描述符 ptm。如果打开失败，则返回一个运行时异常。这个主终端用于与子进程进行通信。

2. 接下来，对主终端进行一些设置。首先，通过 grantpt 函数和 unlockpt 函数对主终端进行授权和解锁。然后，通过 ptsname_r 函数或者 ptsname 函数获取主终端的设备名称 devname。如果授权、解锁或者获取设备名称失败，则返回一个运行时异常。

3. 然后，通过 tcgetattr 函数获取主终端的终端属性，并对属性进行修改。设置属性的输入模式为 UTF-8，同时禁用流控制，以防止按下 Ctrl+S 键锁定显示。最后，通过 tcsetattr 函数将修改后的属性应用到主终端。

4. 设置主终端的窗口大小，使用 struct winsize 结构体和 ioctl 函数实现。窗口大小由参数 rows 和 columns 指定。

5. 调用 fork 函数创建一个子进程。如果创建失败，则返回一个运行时异常。如果是父进程，则将子进程的进程 ID 存储在 pProcessId 指向的变量中，并返回主终端的文件描述符 ptm。如果是子进程，则继续执行后续代码。

6. 在子进程中，首先通过 sigfillset 函数设置一个包含所有信号的信号集 signals_to_unblock，然后通过 sigprocmask 函数将该信号集解除阻塞。这样做是为了清除在 Android Java 进程中可能已经被阻塞的信号。

7. 关闭主终端的文件描述符 ptm，调用 setsid 函数创建一个新的会话，并将子进程设置为会话的首进程。

8. 使用 open 函数打开之前获取到的设备名称 devname 对应的从终端，获取从终端的文件描述符 pts。如果打开失败，则调用 exit 函数退出子进程。

9. 使用 dup2 函数将从终端的文件描述符复制到标准输入、标准输出和标准错误输出的文件描述符上，使得子进程的输入输出与从终端关联。
10. 使用 opendir 函数打开/proc/self/fd 目录，遍历该目录中的文件项，关闭除了标准输入、输出和错误输出以外的所有文件描述符。然后关闭目录。

11. 调用 clearenv 函数清除子进程的环境变量。如果传入了 envp 参数，则通过循环将 envp 指向的环境变量设置到子进程的环境中。

12. 使用 chdir 函数切换子进程的当前工作目录为 cwd 指定的路径。如果切换失败，则通过 perror 函数输出错误信息。

13. 调用 execvp 函数执行指定的命令 cmd，并将 argv 作为命令的参数。如果 execvp 函数返回，说明执行命令失败，通过 perror 函数输出错误信息。

14. 如果到达这里，说明子进程无法执行指定的命令，调用\_exit 函数终止子进程。

可见此创建终端函数就是对前面所讲原理的实现。

此外除了创建终端函数，还提供了一些对终端的基本操作如：

- Java_com_termux_terminal_JNI_setPtyWindowSize：设置终端的窗口大小，这会影响到前端一行和一列的字体个数

- Java_com_termux_terminal_JNI_waitFor：一个阻塞函数，等待 slave 端子进程退出，返回子进程的退出状态码，用于检测终端什么时候退出以及是否正常退出

- Java_com_termux_terminal_JNI_close：关闭终端，释放资源，通过 close 掉前面 open 得到的 ptm 描述符，即可关闭终端释放 fork 的子进程

一个值得注意的地方：Termux 将对终端的读写操作封装在了 java 中，通过在 C 中返回的 ptm 描述符在 java 去进行读写。

### 一个简单的自我实现

好了，分析了 Termux 核心原理，接下来就可以自己尝试实现一个简易伪终端了，要求能够实现简单的读写操作。

#### 简单封装一个为终端 Mterm 类

类定义及方法：

```c++
class Mterm
{
    friend class MtermPool;
private:
    int ptmFd_; // 存放master描述符
    int ptsProcessId_; // 存放slave进程的PID
    long long lastReadTime_; // 最后一次读取的时间戳 为了之后封装的MtermPool方便实现超时回收
    bool isRunning_;// 是否在运行，通过UpdateRunning更新
public:
    Mterm();
    ~Mterm();
    Mterm(const Mterm&) = delete;
    Mterm(const Mterm&&) = delete;
    Mterm& operator=(const Mterm&) = delete;
    Mterm& operator=(const Mterm&&) = delete; // 以上四个拷贝构造函数和赋值运算符用于实现对象禁止复制
    int Create
    (
        const char* cmd,
        const char* cwd,
        char* const argv[],
        char** envp,
        unsigned short rows,
        unsigned short cols
    ); // 创建终端
    void Destrory(); // 销毁终端
    void UpdateRunning(); // 更新运行状态
    bool IsRunning() const { return isRunning_; }; // 是否在运行
    int Read(char* buf, unsigned long size); // 读取终端输出
    int Write(const void* buf, unsigned long size) const; // 写入终端输入
    int Wait() const; // 阻塞模式下等待子进程退出，返回子进程的退出状态码，非阻塞模式下返回进程运行状态
    void SetReadNonblock() const; // 设置读取非阻塞模式
    void ResizeWindow(unsigned short rows, unsigned short cols); // 设置伪终端窗口大小
};
```

具体实现部分，这个不用多说了，其实就是对 Termux 的 Copy 和修补完善:

```c++
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/ioctl.h>
#include <sys/wait.h>
#include <fcntl.h>
#include <termios.h>
#include <signal.h>
#include <dirent.h>
#include <unistd.h>
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <chrono>
#include "mterm.h"
#include "error_number.h"

Mterm::Mterm()
    :ptmFd_(-1),
    ptsProcessId_(-1),
    lastReadTime_(-1),
    isRunning_(false)
{
}

Mterm::~Mterm()
{
    if (isRunning_)
    {
        Destrory();
    }
}

int Mterm::Create
(
    const char* cmd,
    const char* cwd,
    char* const argv[],
    char** envp,
    unsigned short rows,
    unsigned short cols
)
{
    int ptm = open("/dev/ptmx", O_RDWR);
    if (ptm < 0) return OPEN_PTMX_ERROR;

    // pts devname
    char devname[64];
    if (grantpt(ptm) || unlockpt(ptm) || ptsname_r(ptm, devname, sizeof(devname)))
        return GET_PTS_ERROR;

    // utf-8 mode
    struct termios tios;
    tcgetattr(ptm, &tios);
    tios.c_iflag |= IUTF8;
    tios.c_iflag &= ~(IXON | IXOFF);
    tcsetattr(ptm, TCSANOW, &tios);

    // init windows
    struct winsize ws = { .ws_row = rows,.ws_col = cols };
    ioctl(ptm, TIOCSWINSZ, &ws);

    // fork
    pid_t pid = fork();
    if (pid < 0)
    {
        return PROCESS_FORK_ERROR;
    }
    else if (pid > 0)
    {
        ptsProcessId_ = (int)pid;
        ptmFd_ = ptm;
        isRunning_ = true;
        return ptm;
    }
    else
    {
        // signal nonblock
        sigset_t signalsToNonblock;
        sigfillset(&signalsToNonblock);
        sigprocmask(SIG_UNBLOCK, &signalsToNonblock, NULL);

        close(ptm);
        setsid();

        int pts = open(devname, O_RDWR);
        if (pts < 0) exit(OPEN_PTS_ERROR);
        // redirect stdin stdout stderr
        dup2(pts, 0);
        dup2(pts, 1);
        dup2(pts, 2);
        // close all other fds
        DIR* dir = opendir("/proc/self/fd");
        if (dir != NULL)
        {
            int dirFd = dirfd(dir);
            struct dirent* entry;
            while ((entry = readdir(dir)) != NULL)
            {
                int fd = atoi(entry->d_name);
                if (fd > 2 && fd != dirFd) close(fd);
            }
            closedir(dir);
        }
        // env
        if (envp != NULL)
        {
            clearenv();
            for (;*envp;++envp) putenv(*envp);
        }
        // cwd
        if (cwd != NULL && chdir(cwd) != 0)
        {
            char* errorMsg;
            if (asprintf(&errorMsg, "chdir(\"%s\")", cwd) != -1)
                errorMsg = (char*)"chdir()";
            perror(errorMsg);
            fflush(stderr);
        }
        // exec
        execvp(cmd, argv);
        //exec failed
        char* errorMsg;
        if (asprintf(&errorMsg, "exec(\"%s\")", cmd) != -1)
            errorMsg = (char*)"exec()";
        perror(errorMsg);
        fflush(stderr);
        exit(PTS_EXEC_ERROR);
    }
}

void Mterm::Destrory()
{
    close(ptmFd_);
    ptmFd_ = -1;
    ptsProcessId_ = -1;
    lastReadTime_ = -1;
    isRunning_ = false;
}

void Mterm::UpdateRunning()
{
    int status;
    int res = waitpid(ptsProcessId_, &status, WNOHANG);
    if (res == 0)
        isRunning_ = true;
    else
        Destrory();
}

int Mterm::Read(char* buf, unsigned long  size)
{
    int res = read(ptmFd_, buf, size);
    if (res > 0)
    {
        std::chrono::system_clock::time_point now = std::chrono::system_clock::now();
        std::chrono::milliseconds timestamp =
            std::chrono::duration_cast<std::chrono::milliseconds>(now.time_since_epoch());
        lastReadTime_ = timestamp.count();
    }
    return res;
}

int Mterm::Write(const void* buf, unsigned long  size) const
{
    return write(ptmFd_, buf, size);
}

int Mterm::Wait() const
{
    int status;
    waitpid(ptmFd_, &status, 0);
    if (WIFEXITED(status)) {
        return WEXITSTATUS(status);
    }
    else if (WIFSIGNALED(status)) {
        return -WTERMSIG(status);
    }
    else {
        return WAIT_PID_ERROR;
    }
}

void Mterm::SetReadNonblock() const
{
    int flag = fcntl(ptmFd_, F_GETFL);
    flag |= O_NONBLOCK;
    fcntl(ptmFd_, F_SETFL, flag);
}

void Mterm::ResizeWindow(unsigned short rows, unsigned short cols)
{
    struct winsize ws = { .ws_row = rows,.ws_col = cols };
    ioctl(ptmFd_, TIOCSWINSZ, &ws);
}
```

## 封装一个伪终端操作库-libmterm

### 为什么需要封装一个库

我觉得主要有以下优点：

- 把大多数操作封装到这个库中，向外只提供创建、销毁、读写等操作，可以方便地在其他不同项目中使用，而不是像 Termux 那样在 java 中实现读写。

- 这个伪终端原理实际上适用于所有类 Unix 操作系统，想封装一个跨平台的操作库，适用于各个类 Unix 操作系统。

- 封装一个 MtermPool 更好的管理终端，实现资源超时回收，限定进程最多同时使用多少个终端，目前实测在 android 上，同一个父进程最多只能创建 8 个伪终端。

- 封装相同的接口，供不同语言调用

### libmterm 项目目录结构

```shell
code01@code01-A34S:~/桌面/project/libmterm$ tree -L 2
.
├── example  # 调用示例，目前支持c/c++/java/rust调用
│   ├── cxx
│   ├── java
│   └── rust
├── include # 头文件定义
│   ├── error_number.h
│   ├── jni_libmterm.h
│   ├── libmterm.h
│   ├── mterm.h
│   ├── mterm_pool.h
│   ├── rwlock.h
│   └── singleton.h
├── jni # 存放 java JNI.h android 和 linux平台略有不同
│   ├── android
│   └── linux
├── rs # rs调用封装库
│   ├── Cargo.lock
│   ├── Cargo.toml
│   ├── lib -> ../output/
│   ├── src
│   └── target
└── src # 实现
    ├── jni_libmterm.cpp
    ├── libmterm.cpp
    ├── mterm.cpp
    └── mterm_pool.cpp
```

### 介绍两个重要的部分

#### MtermPool

对 Mterm 的管理，包括创建、销毁、读写等操作。

MtermPool 是一个单例类，它管理着所有 Mterm 的实例。

当进程调用时只会有一个 MtermPool 实例，从而保证对最多的 终端限制。

实现了长久不读终端的回收。

定义：

```c++
class MtermPool {
    friend class Singleton<MtermPool>;
public:
    enum class CheckStatus {
        FREE,
        UNCHECKED,
        CHECKED
    };
private:
    std::map<unsigned int, Mterm*> mtermMap_;
    std::map<unsigned int, CheckStatus> checkedMap_;
    Rwlock* rwlock_;
    std::thread* gcThread_;
private:
    MtermPool();
    ~MtermPool();
    MtermPool(const MtermPool&) = delete;
    MtermPool(const MtermPool&&) = delete;
    MtermPool& operator=(const MtermPool&) = delete;
    MtermPool& operator=(const MtermPool&&) = delete;
public:
    int CreateMterm
    (
        const char* cmd,
        const char* cwd,
        char* const argv[],
        char** envp,
        unsigned short rows,
        unsigned short cols
    );
    int CreateMterm
    (
        unsigned short rows = 25,
        unsigned short cols = 80
    );
    int DestroyMterm(unsigned int id);
    int ReadMterm(unsigned int id, char* buf, unsigned long size);
    int WriteMterm(unsigned int id, const char* buf, unsigned long size);
    int WaitMterm(unsigned int id);
    void SetReadNonblockMterm(unsigned int id);
    void SetWindowSizeMterm(unsigned int id, unsigned short rows, unsigned short cols);
    bool CheckRunning(unsigned int id);
private:
    bool IsIdValid(unsigned int id);
    int FindFreeMterm();
    void ResetMterm(unsigned int id);
    unsigned int InsertNewMterm();

    void StartGCThread();
};
```

具体实现请见仓库

#### 接口

接口定义在 libmterm.h 中

采用了 C 风格的接口，方便其他语言调用。

定义：

```c++
#ifdef __cplusplus
extern "C"
{
#endif

    int CreateMterm
    (
        const char* cmd,
        const char* cwd,
        char* const argv[],
        char** envp,
        unsigned short rows,
        unsigned short cols
    );
    int CreateMtermDefault();
    int DestroyMterm(unsigned int id);
    int ReadMterm(unsigned int id, char* buf, unsigned long size);
    int WriteMterm(unsigned int id, const char* buf, unsigned long size);
    int WaitMterm(unsigned int id);
    void SetReadNonblockMterm(unsigned int id);
    void SetWindowSizeMterm(unsigned int id, unsigned short rows, unsigned short cols);
    bool CheckRunningMterm(unsigned int id);

#ifdef __cplusplus
}
#endif
```

### 一些调用示例

具体代码请见仓库 example 目录

java/安卓：
![android](/images/mterm/Screenshot_20231211_104037.jpg)

c++/ubuntu:
![c++](/images/mterm/QQ20240120-130548.png)

## 最后

本章节到此结束，下一篇我会讲解[mterm](https://blog.hackerfly.cn/2024/01/19/mterm/mterm)部分，mterm 是用 Tauri、Vue、kotlin 实现的前端部分，实现通过 Tauri 中 Rust 语言的 ffi 调用 libmterm 的接口，最终实现通过 JavaScript 完成对终端的操作
