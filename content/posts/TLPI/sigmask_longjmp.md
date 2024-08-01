---
title: "在信号处理器函数中执行非本地跳转"
date: 2024-07-27
draft: false
tags: ["TLPI", "signals"]
categories: ["Learning"]
---

## TLPI 21.2.1：在信号处理器函数中执行非本地跳转

```c
#define _GNU_SOURCE // 添加该定义以正常使用strsignal()
#include <string.h>
#include <setjmp.h>
#include <signal.h>
#include "signal_functions.h"
#include "tlpi_hdr.h"

static volatile sig_atomic_t canJump = 0;

#ifdef USE_SIGSETJMP
static sigjmp_buf senv;
#else
static jmp_buf env;
#endif

static void handler(int sig) {
    printf("Received signal %d (%s), signal mask is:\n", sig, strsignal(sig));
    printSigMask(stdout, NULL);

    if (!canJump) { // 若canJump为0
        printf("'env' buffer not yet set, doing a simple return\n");
        return;
    }

#ifdef USE_SIGSETJMP
    siglongjmp(senv, 1);
#else
    longjmp(env, 1);
#endif
}

int main(int argc, char *argv[]) {
    struct sigaction sa;
    printSigMask(stdout, "Signal mask at startup:\n");  // 打印最开始的信号掩码
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;
    sa.sa_handler = handler;
    if (sigaction (SIGINT, &sa, NULL) == -1)
        errExit("sigaction");
#ifdef USE_SIGSETJMP
    printf("Calling sigsetjmp()\n");
    /*
        
    */
    if (sigsetjmp(senv, 1) == 0)
#else
    printf("Calling setjmp()\n");
    // 设置跳转目标，并初始化env
    if (setjmp(env) == 0)
#endif
        // 当从处理器函数直接返回之后才会执行该语句
        canJump = 1;
    else 
        // 当从处理器函数执行非本地跳转之后才会执行该语句
        printSigMask(stdout, "After jump from handler, signal mask is:\n");
    
    printf("Will be paused\n");
    for (;;)
        /*
            等待信号，
            若接收到SIGINT，则跳转到处理器函数中，再由longjmp()/siglongjmp()跳转到setjmp()/sigsetjmp()
            具体是哪个跳转函数，则取决于是否宏定义了USE_SIGSETJMP
        */
        pause();
}

```

## 解释

### canjump的用法

接收到信号的时机有两种：

1. 在`setjmp(env)`之前，在这种情况下，跳转目标尚未建立，这将导致处理器函数使用尚未初始化的 env 缓冲区来执行非本地跳转；
2. 在`setjmp(env)`之后，在这种情况下，跳转目标被建立，env已被初始化在执行`longjmp()/siglongjmp()`之后，可以正确地将一些信息保存。

为了避免出现第一种情况，设置了 canJump 变量，当第一次执行了`setjmp(env)`之后canjump被设置为1，代表着可以正确执行非本地跳转。然后在此`Handler()`的`if (canJump)`处，若 canjump 为0，说明接收到信号的时机是在 set 之前，则不能够进行跳转，而是直接从处理器函数返回。

### 两种跳转方法的区别

`sigsetjmp()`比`setjmp()`多出一个参数 savesigs。如果指定 savesigs 为非 0，那么会将调用 sigsetjmp()时进程的当前信号掩码保存于 env 中，之后通过指定相同 env 参数的`siglongjmp()`调用进行恢复。如果 savesigs 为0，则不会保存和恢复进程的信号掩码。
