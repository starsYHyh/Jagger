---
title: "在备选栈中处理信号"
date: 2024-07-31
draft: false
tags: ["TLPI"]
categories: ["Learning"]
---
# 本篇代码对应着TLPI 21.3：在备选栈中处理信号：sigaltstack()

## 代码
```c
static void sigsegvHandler(int sig) {
    int x;
    // 捕捉信号，并通过局部变量的位置来大致判断为当前函数所分配的空间处于什么位置
    printf("Caught signal %d (%s)\n", sig, strsignal(sig));
    printf("Top of handler stack near %10p\n", (void *)&x);
    // fflush(NULL)的作用是在程序异常终止前确保所有标准输出缓冲区的数据都被写入到相应的输出设备。
    fflush(NULL);
    _exit(EXIT_FAILURE);
}

static void overflowStack(int callNum) {
    char a[100000]; // 此类分配数组方式为栈上分配
    printf("Call %4d - top of stack near %10p\n", callNum, &a[0]);
    overflowStack(callNum + 1); // 无限递归调用，不断在栈上分配空间，每次分配100000字节以上
}

int main(int argc, char *argv[]) {
    stack_t sigstack;
    struct sigaction sa;
    int j;

    printf("Top of standard stack is near %10p\n", (void *)&j);

    // 分配备用栈并通知内核
    // 在堆当中分配备用栈
    sigstack.ss_sp = malloc(SIGSTKSZ);
    if (sigstack.ss_sp == NULL)
        errExit("malloc");
    sigstack.ss_size = SIGSTKSZ;    // 默认栈大小
    sigstack.ss_flags = 0;
    // sig alt stack
    if (sigaltstack(&sigstack, NULL) == -1)
        errExit("sigaltstack");
    // sbrk(0)返回当前进程的堆顶地址
    printf("Alternate stack is at %10p-%p\n", sigstack.ss_sp, (char *)sbrk(0) - 1);

    // 设置信号处理器函数，并且不阻塞任何信号
    sa.sa_handler = sigsegvHandler;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = SA_ONSTACK;
    if (sigaction(SIGSEGV, &sa, NULL) == -1)
        errExit("sigaction");
    // 递归调用overflowStack()
    overflowStack(1);
}
```

## 解释
### ulimit
使用`ulimit -s unlimited`会负责移除当前shell会话时设置的任何 RLIMIT_STACK 资源限，对其他shell会话无影响。但是在使用了这句话之后调用程序，系统会一直打印信息至`Call 78522 - top of stack near 0x7ffa472aac80`，即递归调用了78522次，然后`[1]    16044 killed     ./T_SIGALTSTACK`直接将进程杀死。并没有出现预期的`Caught signal 11 (Segmentation fault)`。

于是我将限制资源回复成了默认大小，再进行调用，出现了预期的结果：
```bash
Top of standard stack is near 0x7ffd97398ebc
Alternate stack is at 0x55fb092fd6b0-0x55fb0931dfff
Call    1 - top of stack near 0x7ffd973807e0
Call    2 - top of stack near 0x7ffd97368110
...
Call   83 - top of stack near 0x7ffd96bad940
Caught signal 11 (Segmentation fault)
Top of handler stack near 0x55fb092ff164
```

经过查阅资料，当使用`ulimit -s unlimited`来移除栈大小限制时，程序可以无限制地使用栈空间。在你提供的代码中，`overflowStack()`函数通过递归调用，不断分配大量内存（每次调用分配100000字节），很快会耗尽系统的可用内存。
当系统检测到内存耗尽时，会启动OOM Killer来终止占用大量内存的进程。在你的例子中，程序在递归调用了78522次后，占用了大量内存，触发了OOM Killer，直接将进程终止，这就是为什么进程被直接杀死而不是触发SIGSEGV信号处理程序。

### fflush(NULL)
关于`fflush(NULL)`，是为了在程序异常终止前确保所有标准输出缓冲区的数据都被写入到相应的输出设备。尽管程序有固定的执行顺序，但标准输出（stdout）通常是缓冲的。**这意味着输出的数据并不会立即被写入到屏幕或文件，而是先存储在缓冲区中**，直到缓冲区满或者遇到刷新操作（如换行、fflush调用等）才会被真正写出。

在正常情况下，`printf`输出的数据会在适当的时机刷新到屏幕上。但是，如果程序异常终止，缓冲区中的数据可能没有机会被刷新，从而导致部分或全部输出丢失。`exit` 函数会执行标准库的清理操作，包括刷新缓冲区。但 `_exit` 函数是直接退出，不进行任何清理操作，包括缓冲区的刷新。这也就是为什么示例程序在`_exit()` 之前要进行`fflush()` 操作。
