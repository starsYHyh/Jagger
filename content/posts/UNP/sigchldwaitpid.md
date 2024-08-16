---
title: "在备选栈中处理信号"
date: 2024-07-31
draft: false
tags: ["UNP", "tcpcliserv"]
categories: ["Learning"]
---

## 处理SIGCHLD信号

```c
#include "sigchldwait.h"

void sig_chld(int signo) {
    pid_t pid;
    int stat;

    // wait()是为了清理僵死进程
    // pid = wait(&stat);
    // printf("child %d terminated\n", pid);

    while ((pid = waitpid(-1, &stat, WNOHANG)) > 0) 
        printf("child %d terminated\n", pid);
    return;
}
```

这段代码定义了一个信号处理函数 `sig_chld(int signo)`，用于处理 `SIGCHLD` 信号，该信号会在一个子进程终止时发送给父进程。让我们逐步解释这段代码的各个部分：

### 1. **函数原型与信号处理**

```c
void sig_chld(int signo);
```

- 这是一个信号处理函数，用来处理接收到的 `SIGCHLD` 信号。`signo` 是传递给处理函数的信号编号，在这种情况下，它的值通常是 `SIGCHLD`。

### 2. **定义变量**

```c
pid_t pid;
int stat;
```

- `pid`：用于存储子进程的进程ID。
- `stat`：用于存储子进程的终止状态。

### 3. **清理僵尸进程**

```c
while ((pid = waitpid(-1, &stat, WNOHANG)) > 0)
    printf("child %d terminated\n", pid);
```

- `waitpid(-1, &stat, WNOHANG)`：这部分代码是关键。`waitpid` 函数用于等待子进程的状态改变。

  - `pid = -1`：表示等待任何子进程。
  - `&stat`：是一个指向 `int` 的指针，用于存储子进程的终止状态。
  - `WNOHANG`：这个选项告诉 `waitpid` 非阻塞运行。如果没有子进程终止，`waitpid` 将立即返回，而不是阻塞等待。

- `while` 循环：这个循环会一直执行，直到没有子进程终止为止。`waitpid` 返回值会是：
  - 大于 0 的值：表示终止的子进程的 PID，表明有一个子进程结束。
  - 0：表示没有子进程终止。
  - -1：表示没有更多的子进程或者调用失败。

- `printf("child %d terminated\n", pid);`：每当 `waitpid` 成功返回一个子进程的PID时，程序会打印出该子进程的ID。

### 4. **函数返回**

```c
return;
```

- `return;` 结束函数执行，返回到信号处理函数的调用者（通常是内核）。

### **总结**

这段代码实现了一个信号处理函数 `sig_chld`，用于处理 `SIGCHLD` 信号。当一个子进程终止时，父进程会收到 `SIGCHLD` 信号，触发这个处理函数。该函数使用 `waitpid` 结合 `WNOHANG` 选项来处理所有终止的子进程，并避免产生僵尸进程。僵尸进程会占用系统资源，因此在接收到 `SIGCHLD` 信号时及时清理子进程的退出状态是很重要的。

这种处理方式确保了父进程可以继续处理多个子进程的终止，而不会因为某个子进程的终止而导致阻塞，从而避免产生僵尸进程。
