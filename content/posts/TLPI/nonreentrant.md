---
title: "信号处理函数中调用不可重入的函数"
date: 2024-08-01
draft: false
tags: ["TLPI", "signals"]
categories: ["Learning"]
---

## TLPI 21.1.2: 可重入函数和异步信号安全函数

## 代码

```c
#define _XOPEN_SOURCE 600
#include <unistd.h>
#include <signal.h>
#include <string.h>
#include <crypt.h> 
#include "tlpi_hdr.h"

static char *str2;
static int handled = 0;

// 设置处理器函数，一旦遇上SIGINT，便对str2进行加密
static void handler(int sig) {
    crypt(str2, "xx");
    handled++;
}

int main(int argc, char *argv[]) {
    char *cr1;
    int callNum, mismatch;
    struct sigaction sa;

    if (argc != 3) 
        usageErr("%s str1 str2\n", argv[0]);

    str2 = argv[2];

    // 对str1进行加密，并将结果保存到独立缓冲区中
    cr1 = strdup(crypt(argv[1], "xx"));

    if (cr1 == NULL)
        errExit("strdup");
    sigemptyset(&sa.sa_mask);   // 初始化在执行处理器函数时将阻塞的信号
    sa.sa_flags = 0;    // 即sa.sa_flags = SIG_BLOCK
    sa.sa_handler = handler;    // 设置信号处理器函数的地址

    if (sigaction(SIGINT, &sa, NULL) == -1)
        errExit("sigaction");


    for (callNum = 1, mismatch = 0; ; callNum++) {
        if (strcmp(crypt(argv[1], "xx"), cr1) != 0) {
            mismatch++;
            printf("Mismatch on call %d (mismatch=%d handled=%d)\n", callNum, mismatch, handled);
        }
    }
}
```

## 解释

### `crypt()`

需要加入`#include <crypt.h>`，如果不加上该头文件，会出现`implicit declaration of function ‘crypt’`的错误。

针对于undefined reference to `crypt'的错误，stackoverflow有以下方案：
> crypt.c:(.text+0xf1): undefined reference to 'crypt' is a linker error. Try linking with `-lcrypt` : gcc crypt.c -lcrypt.

即，在CMakeLists.txt文件中加入`link_libraries(crypt)`。

### for循环内部

在这个无限循环中，不断对str1进行加密，并与正确结果进行比较。如果str1在加密的过程中不被打断，则会得到正确结果，但是如果被打断并在处理器函数中对str2进行加密，则会对str1加密的结果产生污染，导致该加密的结果不正确，以此来判断信号处理器函数是否对不可重入函数产生了影响。
