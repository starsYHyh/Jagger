---
title: "硬件产生的信号"
date: 2024-09-07
draft: false
tags: ["TLPI", "signals"]
categories: ["Learning"]
---

## TLPI 22.4：处理由硬件产生的信号

## 代码（为网站上获取）

```c
#define _GNU_SOURCE     /* Get strsignal() declaration from <string.h> */
#include <string.h>
#include <signal.h>
#include <stdbool.h>
#include "tlpi_hdr.h"

/*
    SIGFPE信号处理器函数
*/
static void sigfpeCatcher(int sig)
{
    printf("Caught signal %d (%s)\n", sig, strsignal(sig));
    sleep(1);                   /* Slow down execution of handler */
}

int main(int argc, char *argv[])
{
    /*
        参数有三种选择，
        第一种，无参数，为捕获信号并进入处理器函数
        第二种，-i，为忽略信号
        第三种，-b，为阻塞信号
    */
    if (argc > 1 && strchr(argv[1], 'i') != NULL) { // strchr用于查找第一次出现'i'的位置，并返回以其为首的字符串
        printf("Ignoring SIGFPE\n");
        if (signal(SIGFPE, SIG_IGN) == SIG_ERR)    // i-忽略SIGFPE
             errExit("signal");
    } else {
        printf("Catching SIGFPE\n");

        struct sigaction sa;
        sigemptyset(&sa.sa_mask);   // 不阻塞任何信号
        sa.sa_flags = SA_RESTART;   // 自动重启
        sa.sa_handler = sigfpeCatcher;
        if (sigaction(SIGFPE, &sa, NULL) == -1)
            errExit("sigaction");
    } 
        
    bool blocking = argc > 1 && strchr(argv[1], 'b') != NULL;   // b-阻塞信号
    sigset_t prevMask;
    if (blocking) {
        printf("Blocking SIGFPE\n");

        sigset_t blockSet;
        sigemptyset(&blockSet);
        sigaddset(&blockSet, SIGFPE);
        if (sigprocmask(SIG_BLOCK, &blockSet, &prevMask) == -1)
            errExit("sigprocmask");
    }

    printf("About to generate SIGFPE\n");   // 准备生成SIGFPE
    int x, y;
    y = 0;
    x = 1 / y;  // 除零操作，生成SIGFPE
    y = x;      /* Avoid complaints from "gcc -Wunused-but-set-variable" */

    if (blocking) {
        printf("Sleeping before unblocking\n");
        sleep(2);
        printf("Unblocking SIGFPE\n");
        if (sigprocmask(SIG_SETMASK, &prevMask, NULL) == -1)
            errExit("sigprocmask");
    }

    printf("Shouldn't get here!\n");
    exit(EXIT_FAILURE);
}
```

## 解释

根据参数，进行后续步骤：

+ 若无参数，则直接为其建立信号处理器函数，内容为打印出相关信息；
+ 若为`-i`，则为忽略 SIGFPE 信号，使用`signal(SIGFPE, SIG_IGN)`这行代码来忽略浮点数异常；
+ 若为`-b`，则为阻塞信号，先将 SIGFPE 放入信号掩码，然后在错误的浮点数运算之后取消阻塞

## 运行结果

### 默认情况

默认情况下：

```bash
Catching SIGFPE
About to generate SIGFPE
Caught signal 8 (Floating point exception)
Caught signal 8 (Floating point exception)
Caught signal 8 (Floating point exception)
Caught signal 8 (Floating point exception)
Caught signal 8 (Floating point exception)
Caught signal 8 (Floating point exception)
Caught signal 8 (Floating point exception)
^\[1]    116013 quit       ./DEMO_SIGFPE
```

在除零操作之后，产生异常，产生了 SIGFPE 信号，将其捕获，打印信息，并睡眠1s。
运行结果不断出现信息的原因是，在退出信号处理器函数之后，程序又回到终端出重新运行，在本程序中，将会不断地重新进行除零操作，从而不断产生异常。

### 忽略信号

在这种情况下：

```bash
Ignoring SIGFPE
About to generate SIGFPE
[1]    113466 floating point exception  ./DEMO_SIGFPE -i
```

结果表明，并没有成功忽略该信号，这也验证了 TLPI 中的描述：
> 当由于硬件异常而产生上述信号之一时，Linux 会强制传递
信号，即使程序已经请求忽略此类信号。

### 阻塞信号

在这种情况下：

```bash
Blocking SIGFPE
About to generate SIGFPE
[1]    116212 floating point exception  ./DEMO_SIGFPE -b
```

同样地，也并没有阻塞该信号。
> 始于 Linux 2.6，如果信号遭到阻塞，那么该信号总是会立刻杀死进程，即使进程已经为此信号安装了处理器函数。
