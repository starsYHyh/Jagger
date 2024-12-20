---
title: "真题总结1"
date: 2023-09-24
draft: false
tags: ["408"]
categories: ["Learning"]
---


## 10年

### 平衡二叉树的调整

LL调整
![image-20230924172546237](https://picgo-myblog.oss-cn-beijing.aliyuncs.com/Images/image-20230924172546237.png)

RR调整
![image-20230924172610124](https://picgo-myblog.oss-cn-beijing.aliyuncs.com/Images/image-20230924172610124.png)

RL调整
![image-20230924173015698](https://picgo-myblog.oss-cn-beijing.aliyuncs.com/Images/image-20230924173015698.png)

例如（原谅我图画的难看😂）：  
![image-20230924182109395](https://picgo-myblog.oss-cn-beijing.aliyuncs.com/Images/image-20230924182109395.png)

LR调整
![image-20230924173032244](https://picgo-myblog.oss-cn-beijing.aliyuncs.com/Images/image-20230924173032244.png)

### 二分查找的次数（成功失败）、折半查找判定树与二叉搜索树的关系

   ![image-20230924161402406](https://picgo-myblog.oss-cn-beijing.aliyuncs.com/Images/image-20230924161402406.png)
   ![image-20230924161158871](https://picgo-myblog.oss-cn-beijing.aliyuncs.com/Images/image-20230924161158871.png)
平衡二叉树是一种特殊的二叉查找树，它要求任何节点的两棵子树的高度差不超过1（同一棵树的不同节点的子树高度差可以为1、0、-1）。平衡二叉树通过在插入和删除节点时做旋转来维持树的平衡。  
折半查找判定树是一种特殊的平衡二叉树，它要求更加严格。**同一棵树节点的左子树和右子树的差不能同时存在1和-1**（即为统一向上取整或统一向下取整）。查找时，根据比较的结果折半排除一边的树。折半查找判定树中，**只有最下面一层才可以不满**。
根据完全二叉树的高度计算公式，元素个数为n时，树高 $h=\lceil log_2(n+1)\rceil$ 或 $h=\lfloor log_2(n)\rfloor+1$ 。在折半查找中，

+ 查找成功的最小比较次数为1，最大比较次数为$h$；
+ 查找失败的最小比较次数为：若$n=2^h-1$，则为$h$，否则为$h-1$，最大比较次数：$h$。

### 数据类型转换造成的精度丢失等问题

将高精度数转换为低精度数可能会引起：

1. **精度丢失**：高精度数通常能够表示更大范围和更高精度的数字，但当将其转换为低精度数时，可能会导致小数部分被截断或丢失，从而引起精度丧失。
2. **溢出**：如果高精度数的值超出了低精度数所能表示的范围，会导致溢出。

将整型数转换为浮点数一般不会出现问题，但特殊情况下会导致精度丢失，对于非常大或非常小的整数，可能无法精确表示。如int类型数据二进制表示有32位，但是对于float类型，在IEEE 754格式下，尾数部分只有23位，可能无法完全表示某一个int类型数，造成精度的丢失。
单精度与双精度浮点数的运算也有可能会有一些问题，例如10年真题$T_{14}$，$f=1.5678e3,d=1.5e100$，进行$(d+f)-d$运算，在$d+f$时，需要先进行对阶（小阶向大阶对齐），由于格式中的尾数限制，对阶后，$f$的尾数被舍去而变成了0，故$d+f$仍然为$d$，再减去$d$结果为0，而不是$f$。

### 字扩展、位扩展与相关的芯片最低地址问题

位扩展

位扩展是指用若干片位数较少的存储器芯片构成给定字长的存储器，容量改变，位数改变，地址单元个数不变。

在袁春风老师的《计算机系统基础》中的一个例子如下：  
![image-20230924203735570](https://picgo-myblog.oss-cn-beijing.aliyuncs.com/Images/image-20230924203735570.png)

注意到8个$16M\times8bit$的芯片扩展构成一个128M内存条。在进行扩展之前，对于单个芯片，地址位数为24bit。即地址单元个数为$16M=2^{24}$ 。每个地址单元存储8bit。  
在进行扩展之后，$128M=2^{27}$，而行列地址位数加起来一共$24bit$，$2^{24}=16M$，则说明此次扩展为位扩展。一个地址单元中的数据位数增加，但是**总的地址单元个数并未改变**。

12位行地址$i$和12位列地址$j$分别送到DRAM芯片内部的行地址译码器和列地址译码器，选择行列地址交叉点$(i,j)$的8位数据同时进行读写，**8个芯片就可以同时读取64bit**，组合成总线所需要的64位传输宽度，再通过存储器总线进行传输。

字扩展

字扩展，容量改变，地址单元个数改变，即地址位数会改变，位数不会改变。

另外，对于某一存储器，由多个DRAM经过字扩展而成，那么**对于单个DRAM芯片来说，其行地址RAS位数、列地址CAS位数也不会变，但是整体存储器的总地址位数会变，会增加片选信号位数**。  此时，对于一个主存地址，可以理解为`芯片序号+芯片内的位置` 。
也就是说，无论是位扩展还是字扩展，对于单一的DRAM芯片，其行列地址位数均不含会改变。（14年T15）

### 引起进程状态改变的一些典型事件

进程状态可以在以下情况下发生改变：

1. **创建新进程**：当操作系统启动一个新的程序时，会创建一个新的进程，并将其状态**设置为就绪状态**。

2. **进程等待**：当一个进程等待某些事件发生，例如等待用户输入、等待某个文件就绪等，它的状态会**从运行状态变为阻塞状态**。

3. **时间片用完**：如果一个进程在分配给它的时间片用完之后，调度器会将其状态**从运行状态变为就绪状态**，降低其进程优先级，然后选择下一个要执行的进程。

4. **I/O操作完成**：当一个进程等待的I/O操作完成后，它的状态会**从阻塞状态变为就绪状态**。

5. **进程终止**：当一个进程完成了它的任务，或者由于某种原因需要被终止，它的状态会**从运行状态变为终止状态**。

6. **进程被阻塞的资源可用**：当一个进程等待的资源（如锁或信号量）变为可用时，它的状态会**从阻塞状态变为就绪状态**。设备分配是在一个已经存在的进程中进行的，不会导致创建新进程。

7. **进程被唤醒**：在多任务环境中，一个进程可能会被另一个进程唤醒，使得它**从阻塞状态变为就绪状态**。

8. **进程被挂起或恢复**：操作系统可以将一个进程从内存中挂起（**暂时移出内存**）或者从挂起状态恢复（**重新加载到内存中**）。

9. **父进程等待子进程**：当一个父进程等待其子进程结束时，它的状态可能会**从运行状态变为阻塞状态**。

10. **发生错误或异常**：当一个进程遇到错误或异常情况时，它的状态可能会**从运行状态变为终止状态或者阻塞状态**（如果它在等待某些事件发生时发生了错误）。

### IO系统的分层结构

![IO子系统](https://picgo-myblog.oss-cn-beijing.aliyuncs.com/Images/IO%E5%AD%90%E7%B3%BB%E7%BB%9F.svg)

![image-20230924211859133](https://picgo-myblog.oss-cn-beijing.aliyuncs.com/Images/image-20230924211859133.png)

### 碰撞域、广播域

1. 冲突域：在同一个冲突域中，每一个结点都能收到所有其他结点发送的帧。简单地说，冲突域为同一时间内只能有一台设备发送信息的范围。
2. 广播域：网络中能接收任意设备发出的广播帧的所有设备的集合。即，如果站点发出一个广播信号，所有能接收到这个信号的设备范围被称为一个广播域。

**通常一个网段为一个冲突域，一个局域网为一个广播域。**

### 磁盘的调度策略

磁盘调度算法是操作系统中用于管理磁盘上的I/O请求的一种策略。以下是一些常见的磁盘调度算法：

1. **先来先服务（FCFS）**：最简单的磁盘调度算法，按照请求的到达顺序依次执行。但可能会出现“早来的请求等待时间长”的问题。
2. **最短寻道时间优先（SSTF）**：选择当前磁头位置最近的请求进行服务，以最小化寻道时间。但可能会导致某些请求长时间等待。
3. **SCAN算法**：扫描算法，也称为电梯算法，类似于电梯的运行方式，磁头按一个方向移动，直到最后一个磁道后**再改变方向**。
4. **C-SCAN算法**：循环扫描算法，磁头按同一个方向移动，直到**到达最后一个磁道后**，立即直接返回到磁道0处，再继续扫描。
5. **LOOK 算法**：类似于扫描算法，但在到达最远的请求后不会立即返回，而是**根据当前请求的方向决定下一个服务的磁道**。在朝一个给定方向移动前查看是否有请求。
6. **C-LOOK 算法**：类似于 C-SCAN 算法，但在到达最远的请求后不会立即返回，而是**直接返回到最远端的有请求的磁道**。

注意，在做题时，若无特别说明，也可以默认SCAN算法和C-SCAN算法为LOOK和C-LOOK调度。

## 09年

### *B树与B+树的异同*

B树和B+树都是一种常用的多路搜索树数据结构，用于存储有序的数据集合，特别是在磁盘存储和数据库系统中应用广泛。

1. **数据存储**：
   + **B树**：B树的叶子节点和非叶子节点的结构基本相同，都可以存储数据。
   + **B+树**：在B+树中，只有叶子节点存储数据，非叶子节点只存储索引（有点像文件管理中的索引顺序文件），叶子节点之间通过指针连接，形成一个有序的链表。
2. **范围查询效率**：
   + **B树**：由于B树的叶子节点也存储数据，因此可以直接进行范围查询。
   + **B+树**：由于B+树的非叶子节点不存储数据，只有叶子节点存储数据，因此范围查询需要在叶子节点构成的链表上进行。
3. **查询性能**：
   + **B树**：由于B树的非叶子节点也存储数据，单次查询可能直接在非叶子节点找到结果，因此查询性能相对于B+树可能更快。
   + **B+树**：B+树的查询性能相对于B树可能稍慢，因为每次查询都需要在叶子节点链表上进行。

![image-20230925190840348](https://picgo-myblog.oss-cn-beijing.aliyuncs.com/Images/image-20230925190840348.png)

### 浮点数加减的步骤（IEEE 754）

+ 正常方法：对阶（小阶向大阶对齐）、尾数运算、规格化（左规、右规）、舍入、判断溢出（判断舍入后的阶数是否超过了所能表示的范围）；

+ 偷懒方法：直接计算（按照大的阶数）、在规格化后判断是否溢出（判断阶数是否超过了所能表示的范围）（如10年T13）

### CISC与RISC的区别

CISC 指令系统设计的主要特点如下：

1. **指令系统复杂且多**。指令条数多，寻址方式多，指令格式多而复杂，指令长度可变，操作码长度可变。
2. **各种指令都能访问存储器**，有些指令还需要多次访问存储器。
3. **不利于实现流水线**。因为指令周期长且差距大。绝大多数指令需要多个时钟周期才能完成，简单指令和复杂指令所用的时钟周期数相差很大。
4. 复杂指令系统使得其实现越来越复杂，不仅增加了研制周期和成本，而且难以保证正确性，甚至因为指令太复杂而无法采用硬连线路控制器，**导致大多数系统只能采用慢速的微程序控制器。**
5. **难以进行编译优化**。由于编译器可选指令序列增多，使得目标代码组合增加，从而增加了目标代码优化的难度。
6. 相关指令会产生显式的条件码，存放在专门的标志寄存器（或称状态寄存器）中，可用于条件转移和条件传送等指令。

相较于CISC，RISC 指令系统设计的主要特点如下：

1. **指令数目少且规整。**只包含使用频度高的简单指令，寻址方式少，指令格式少，指令长度一致，指令中操作码和寄存器编号等位置固定，便于取指令、指令译码以及提前读取寄存器中操作数等。
2. **采用 load/store 型指令设计风格**。一条指令的执行阶段最多只有一次存储器访问操作。
3. **采用流水线方式执行指令**。规整的指令格式有利于采用流水线方式执行，除 load/store指令外，其他指令都只需一个或小于一个时钟周期就可完成，指令周期短。
4. **采用硬连线路控制器**。指令少而规整使得控制器的实现变得简单，可以不用或少用微程序控制器。
5. 指令数量少，固然使编译工作量加大，但由于指令系统中的指令都是精选的，编译时间少，**反过来对编译程序的优化又是有利的。**
6. **采用大量通用寄存器**。编译器可将更多的局部变量分配到寄存器中，并且在过程调用时通过寄存器进行参数传递而不是通过栈进行传递，以减少访存次数。

![image-20230926153615886](https://picgo-myblog.oss-cn-beijing.aliyuncs.com/Images/image-20230926153615886.png)

### 硬布线控制器与微程序控制器的对比

|      | 硬布线控制器 | 微程序控制器 |
| :----: | :----------: | :------------: |
| 实现方式 | 通过硬件电路来实现的，通常是由逻辑门、寄存器和触发器等组件组成。 | 使用一组存储在控制存储器中的微指令序列来实现。 |
| 性能 | 硬件实现，因此执行速度通常非常快 | 略慢一些，每个微指令的执行需要额外的时钟周期。 |
| 复杂性 | 较高 | 相对更为简单 |
| 灵活性 | 控制逻辑固定在硬件中，不容易进行修改或升级，缺乏灵活性。 | 可以通过修改控制存储器中的微指令来改变控制逻辑，灵活性较高。 |

### 文件访问控制

1. **口令保护**
   + 为文件设置一个“口令“，用户想要访问文件时需要提供口令，由系统验证口令是否正确。
   + 实现开销小，但“口令“一般存放在**FCB或索引结点中**（也就是存放在系统中）因此不太安全。
2. **加密保护**
   + 用一个“密码“对文件加密，用户想要访问文件时，需要提供相同的“密码“才能正确的解密。
   + 安全性高，即使文件被非法获取，没有密码也无法正确读取其中的内容。但加密/解密需要耗费一定的时间。
3. **访问控制**
   + 用一个访问控制表 (ACL) 记录各个用户（或各组用户）对文件的访问权限，对文件的访问类型可以分为，读写/执行/删除等。
   + 实现灵活，可以实现复杂的文件保护功能，**存放文件访问控制信息的合理位置为FCB或索引节点。**

### 软硬链接的区别

1. **硬链接**
   硬链接的实现只是在要创建链接的目录中创建了另一个名称，并将其指向**原有文件的相同inode号（即低级别名称）**。该文件不以任何方式复制。在创建文件的硬链接之后，在文件系统中，原有文件名（file）和新创建的文件名（file2）之间没有区别。实际上，它们都只是指向文件底层元数据的链接，**可以在同一个inode中找到**。
   硬链接有个局限在于：不能创建目录的硬链接（因为担心会在目录树中创建一个环）。不能硬链接到其他磁盘分区或文件系统中的文件（因为 inode 号在特定文件系统中是唯一的，而不是跨文件系统）等等。*注意：在Windows操作系统中，磁盘分区通常是以盘符（如C盘、D盘等）来表示的，但这**并不等同于Unix或类Unix系统中的文件系统中的概念**。如果C盘和D盘属于同一物理磁盘（硬盘），可以在它们之间创建硬链接。然而，**如果它们代表了两个不同的物理磁盘**，那么在它们之间创建硬链接通常是不可能的。*
2. **符号链接（软链接）**
   因此，人们创建了一种称为符号链接（软链接）的新型链接。符号链接本身实际上是一个不同类型的文件。**除了常规文件和目录之外，符号链接是文件系统知道的第三种类型**。因为形成符号链接的方式是指向文件的路径名作为链接文件的数据，所以符号链接文件的大小于所指向的文件的路径名的长度有关系，且有可能造成所谓的悬空引用。

### 奈奎斯特定理和香农定理

奈奎斯特定理规定，为了避免信号在数字化过程中产生失真，需要以至少**两倍于信号最高频率的采样率**对信号进行采样。

### FTP

在FTP中，数据传输可以采用两种模式：主动模式和被动模式。

**主动模式**：

1. 客户端向FTP服务器的默认数据端口（通常是端口20）发起连接请求，请求建立数据连接。
2. 服务器在接收到客户端的连接请求后，会从自己的数据端口（通常是端口20）向客户端的随机端口发起连接请求，请求建立数据连接。（服务器发起数据连接请求）

**被动模式**：

1. 客户端向FTP服务器的命令端口（通常是端口21）发送PASV命令，请求进入被动模式。
2. 服务器收到PASV命令后，会打开一个随机的端口，监听客户端的连接请求，同时将该端口号发送给客户端。
3. 客户端收到服务器返回的端口号后，会从自己的端口向服务器的指定端口发起连接请求，请求建立数据连接。（客户端发起数据连接请求）

## 11年

### 计算机性能评价相关词条

### IEEE 754格式化

IEEE 754单精度浮点数的解释

|     值的类型     |  符号  |    阶码    |  尾数   |        值         |
| :--------------: | :----: | :--------: | :-----: | :---------------: |
|       正零       |  $0$   |    $0$     |   $0$   |        $0$        |
|       负零       |  $1$   |    $0$     |   $0$   |       $-0$        |
|     正无穷大     |  $0$   | $255(全1)$ |   $0$   |     $\infty$      |
|     负无穷大     |  $1$   | $255(全1)$ |   $0$   |     $-\infty$     |
| 无定义数（非数） | $0或1$ | $255(全1)$ | $\ne0$  |   $\text{NaN}$    |
|  规格化非零正数  |  $0$   | $0<e<255$  |   $f$   | $2^{e-127}(1.f)$  |
|  规格化非零负数  |  $1$   | $0<e<255$  |   $f$   | $-2^{e-127}(1.f)$ |
|   非规格化正数   |  $0$   |    $0$     | $f\ne0$ |  $2^{126}(0.f)$   |
|   非规格化负数   |  $1$   |    $0$     | $f\ne0$ |  $-2^{126}(0.f)$  |

IEEE 754的范围

| 格式 | 最小值 | 最大值 |
| :----: | :----: | :----: |
| 单精度 | $E=1,M=0$<br>$1.0\times2^{1-127}=2^{-126}$ | $E=254,M=.111\cdots,1.111\cdots1\times2^{254-127}=2^{127}\times(2-2^{-23})=2^{128}-2^{104}$ |
| 双精度 |  $E=1,M=0$<br>$1.0\times2^{1-1023}=2^{-1022}$  | $E=2046,M=.111\cdots,1.111\cdots1\times2^{2046-1023}=2^{1023}\times(2-2^{-52})=2^{1024}-2^{971}$ |

### 加减法法电路

电路

![image-20230930110104653](https://picgo-myblog.oss-cn-beijing.aliyuncs.com/Images/image-20230930110104653.png)

标志位

![image-20230930104238889](https://picgo-myblog.oss-cn-beijing.aliyuncs.com/Images/image-20230930104238889.png)

### 系统总线的各条线传输的是什么

![image-20230928225351372](https://picgo-myblog.oss-cn-beijing.aliyuncs.com/Images/image-20230928225351372.png)

在取指令时，指令便是在数据线上传输的。操作数显然在数据线上传输。中断类型号用以指出中断向量的地址，CPU响应中断请求后，将**中断应答信号 (INTR) 发回到数据总线**上，CPU从数据总线上读取中断类型号后，查找中断向量表，找到相应的中断处理程序入口。而握手（应答）信号属于通信联络控制信号，应在通信总线上传输。

### *OSI模型与TCP/IP模型各层的功能区别*

**OSI模型**：

1. **物理层**：解决使用何种信号来传输比特0和1的问题。负责定义物理媒介、传输速率、编码方法等，提供了物理介质上的原始数据传输。
2. **数据链路层**：解决帧在一个网络（或一段链路）上传输的问题。负责数据帧的传输、接收、错误检测和纠正，确保数据在物理层上可靠地传输。
3. **网络层**：解决分组在多个网络之间传输（路由）的问题。负责在不同网络之间进行路由选择，进行分组的转发和寻址，以及定义子网等网络层面的逻辑结构。
4. **传输层**：解决进程之间基于网络的通信问题。提供端到端的通信，保证数据的可靠传输，处理数据的分段和重组，以及流量控制和拥塞控制。
5. **会话层**：解决进程之间进行会话的问题。建立、管理和终止会话连接，提供了通信会话的控制和同步功能。
6. **表示层**：解决通信双方交换信息的表示问题。负责数据的格式转换、加密解密、压缩解压缩等，保证数据的可靠传输。
7. **应用层**：解决通过应用进程之间的交互来实现特定网络应用的问题。为用户提供各种网络应用服务，如文件传输、电子邮件、远程登录等。

**TCP/IP模型**：

1. **应用层**：与OSI模型的应用层类似，提供网络应用服务。

2. **传输层**：与OSI模型的传输层类似，负责提供端到端的通信服务。

3. **网络层**：与OSI模型的网络层类似，负责在不同网络之间进行路由选择。

4. **链路层**：包括了OSI模型的物理层和数据链路层的功能，负责在相邻节点间提供可靠的数据传输。

在实际应用中，TCP/IP模型是目前广泛使用的网络模型，而OSI模型在理论上用于描述通信系统的功能。

### 对正确接收到的数据帧进行确认的MAC协议

为什么是CSMA/CA协议？

CSMA/CA协议，是一种用于在无线网络中进行数据通信的协议，针对无线环境中信道容量有限、易受干扰的特点而设计。它采用了一些机制来避免碰撞并提高数据传输的可靠性：

1. **监听信道**：发送方在发送数据前会先监听信道，确保信道是空闲的，避免发生碰撞（减少概率，不能一定避免）。

2. **RTS/CTS帧**：在CSMA/CA中，还引入了请求发送（Request to Send，RTS）和清除发送（Clear to Send，CTS）帧的概念。发送方在发送数据前会先发送一个RTS帧，接收方如果收到并确认了RTS帧，就会发送一个CTS帧给发送方，表示信道已经为发送方保留。这个过程可以有效避免隐藏节点问题和减少碰撞的发生。

3. **等待确认**：发送方在发送数据后会等待接收方的确认，只有在收到确认后才会发送下一个数据帧。

4. **重传机制**：如果发送方没有收到确认，会认为数据帧可能丢失，会进行重传。

### *特殊的IP及其作用、使用方法*

![image-20230928201049572](https://picgo-myblog.oss-cn-beijing.aliyuncs.com/Images/image-20230928201049572.png)

以下是RFC文档中的内容[Special-Use IPv4 Addresses RFC 3330](https://datatracker.ietf.org/doc/rfc3330/)

```text
   Address Block             Present Use                       Reference
   ---------------------------------------------------------------------
   0.0.0.0/8            "This" Network                 [RFC1700, page 4]
   10.0.0.0/8           Private-Use Networks                   [RFC1918]
   14.0.0.0/8           Public-Data Networks         [RFC1700, page 181]
   24.0.0.0/8           Cable Television Networks                    --
   39.0.0.0/8           Reserved but subject
                           to allocation                       [RFC1797]
   127.0.0.0/8          Loopback                       [RFC1700, page 5]
   128.0.0.0/16         Reserved but subject
                           to allocation                             --
   169.254.0.0/16       Link Local                                   --
   172.16.0.0/12        Private-Use Networks                   [RFC1918]
   191.255.0.0/16       Reserved but subject
                           to allocation                             --
   192.0.0.0/24         Reserved but subject
                           to allocation                             --
   192.0.2.0/24         Test-Net
   192.88.99.0/24       6to4 Relay Anycast                     [RFC3068]
   192.168.0.0/16       Private-Use Networks                   [RFC1918]
   198.18.0.0/15        Network Interconnect
                           Device Benchmark Testing            [RFC2544]
   223.255.255.0/24     Reserved but subject
                           to allocation                             --
   224.0.0.0/4          Multicast                              [RFC3171]
   240.0.0.0/4          Reserved for Future Use        [RFC1700, page 4]
```

## 12年

### 上三角矩阵和拓扑排序

**若用邻接矩阵存储有向图，矩阵中主对角线及以下的元素均为零（或主对角线及以上的元素均为零），则该图的拓扑序列一定存在，但不一定唯一。**

例如，如果一个有向图使用邻接矩阵进行存储，并且矩阵中主对角线及以下的元素均为零，那么这个图的拓扑序列一定存在。因为拓扑序列是一个图中所有顶点的线性排序，**使得对于每一条有向边 <u,v>，顶点 u 在序列中都出现在顶点 v 之前**。在这种情况下，由于主对角线以下的元素都是零，表明没有顶点指向自己，也就不存在环路，因此必然存在拓扑排序。

然而，拓扑序列可能不是唯一的。这是因为在一个有向无环图中，**可能存在多个顶点没有前驱（即入度为零）**，这些顶点的相对顺序可以任意排列，只要它们在拓扑排序中排在其他顶点之前即可。因此，可能存在多个合法的拓扑序列。

**反之，如果存在拓扑序列，邻接矩阵不一定满足主对角线及以下的元素均为零（或主对角线及以上的元素均为零）。**

### 数据线、地址线、控制线

参考[【组成原理·总线】总线的概念和计算](https://www.cnblogs.com/Mount256/p/16696786.html#121-%E5%8D%95%E6%80%BB%E7%BA%BF%E7%BB%93%E6%9E%84)

在单总线结构中：

1. **数据线**：用于在各个设备（**包括CPU、主存、外设**）之间传递数据。

2. **地址线**：传输地址信息，用于指示要访问的特定**内存单元或外设**的地址。地址线的数量决定了总线的寻址能力，也就是可以寻址的内存或设备数量。

3. **控制线**：传输各种控制信号，如**读取、写入、使能**等信号，以控制数据的流动和操作。控制线**还可能包括时钟信号**等。也可以传输主存返回 CPU 的反馈信号。

在IO接口中：

1. **数据线**：可以传输IO接口中的命令字、状态字、中断类型号，在接口中的数据缓冲寄存器、内存、CPU中的寄存器之间进行数据传送。**（可双向传输）**同时接口和设备的状态信息被记录在状态寄存器中，通过数据线将状态信息送到 CPU。CPU 对外设的控制命令也通过数据线传送。
2. **地址线**：接口中的地址线用于给出要访问的 I/O 接口中的**寄存器的地址**，它和读/写控制信号一起被送到 I/O 接口的控制逻辑部件。(只能单向)
3. **控制线**：通过控制线传送来的**读/写信号**确认是读寄存器还是写寄存器，此外控制线还会传送一些**仲裁信号和握手信号**。（只能单向）

### 中断处理和子程序调用的区别

1. **触发方式**：
   + **中断处理**：中断是由**硬件或外部事件触发**的。当发生了某个预定义的事件（如设备输入、时钟周期等），硬件会暂停当前程序的执行，并跳转到相应的中断处理程序中执行。
   + **子程序调用**：子程序调用是由**程序内部通过软件指令来触发**的。程序会通过调用指令跳转到指定的子程序中执行，执行完后再返回到原来的程序。
2. **上下文切换**：
   + **中断处理**：中断处理涉及到从用户态切换到内核态，然后再返回用户态的过程。这涉及到硬件和操作系统的支持，**需要保存和恢复相应的寄存器和状态，如断点（PC）、程序状态字寄存器（PSW）**。（中断处理中最重要的两个寄存器）
   + **子程序调用**：子程序调用是在同一程序执行流中完成的，没有涉及到从用户态到内核态的切换。因此，它比中断处理的上下文切换开销要小，**只需要保存程序断点**。
3. **异步性质**：
   + **中断处理**：中断是异步的，它可以**随时发生**并打断当前的程序执行。因此，中断处理必须能够处理不可预知的事件。
   + **子程序调用**：子程序调用是同步的，它是由程序内部**明确地发起**的，执行完子程序后会继续执行后续的指令。
4. **权限级别**：
   + **中断处理**：中断处理通常需要在内核态执行。
   + **子程序调用**：子程序可以在用户态或内核态中执行，具体取决于程序的权限和需要。

### 物理接口各个特性的功能

1. 机械特性：指明接口所用接线器的形状和尺寸、引脚数目和排列、固定和锁定装置等。
2. 电气特性：指明在接口电缆的各条线上出现的电压的范围。
3. 功能特性：指明某条线上出现的某一电平的电压表示何种意义。
4. 过程特性：或称规程特性。指明对于不同功能的各种可能事件的出现顺序。

### 协议的三部分

协议由语法、语义和同步三部分组成。

+ 语法规定了传输数据的格式；
+ 语义规定了所要完成的功能，即需要发出何种控制信息、完成何种动作及做出何种应答；
+ 同步规定了执行各种操作的条件、时序关系等，即事件实现顺序的详细说明。

一个完整的协议通常应具有线路管理（建立、释放连接）、差错控制、数据转换等功能。**协议并不关心具体如何实现，只要求实现什么功能。**

### *各层协议有无连接、可不可靠？*

![image-20231001144634286](https://picgo-myblog.oss-cn-beijing.aliyuncs.com/Images/image-20231001144634286.png)

1. **数据链路层**：
     + **以太网的MAC协议提供的是无连接不可靠服务**：考虑到**局域网信道质量好**，以太网采取了两项重要的措施以使通信更简便：采用无连接的工作方式；不对发送的数据帧进行编号，也不要求对方发回确认。因此，以太网提供的服务是不可靠的服务，即尽最大努力的交付。差错的纠正由高层完成。
     + **无线局域网的MAC协议提供的是无连接可靠服务**：并没有“握手”的过程，即传输数据之前并没有连接。

2. **网络层**：
   + **IP协议提供的是无连接不可靠服务**：IP协议不需要在传输数据前建立连接，也不需要在传输过程中保持连接状态。每个IP数据包都是独立传输的，它们之间没有直接的关联。IP数据包的传输是不可靠的，这意味着它们可能会丢失、延迟、重复或者乱序。IP协议不提供任何机制来保证数据的可靠性，也不负责处理丢失的数据包。
3. **传输层**：
   + **TCP协议提供的是有连接可靠服务**：通过序号、确认、重传等机制来保证数据的完整性和有序性。
   + **UDP协议提供的是无连接不可靠服务**：将数据分割成报文段，并将报文段发送到目标，无需建立连接或维护会话状态。但是可以通过**在应用层实现自定义的可靠性机制**，使得UDP的通信变得可靠。

### *协议A直接为协议B提供服务*

![image-20231009204458440](https://picgo-myblog.oss-cn-beijing.aliyuncs.com/Images/image-20231009204458440.png)

| 协议A | 协议B |
| :---: | :---: |
|  TCP  |  BGP  |
|  IP   | OSPF  |
|  UDP  |  RIP  |
|  IP   | ICMP  |
|  IP   |  TCP  |
|  IP   |  UDP  |
|  UDP  |  DNS  |
|  TCP  | HTTP  |
|  TCP  | SMTP  |
|  TCP  | POP3  |
|  IP   | IGMP  |
|  TCP  |  FTP  |
|       |       |
|       |       |
|       |       |

## 13年

### 最佳归并树

![image-20231004174943348](https://picgo-myblog.oss-cn-beijing.aliyuncs.com/Images/image-20231004174943348.png)

有n个初始归并段，需要做k路平衡归并排序，可以根据哈夫曼树的思想来优化WPL。

### RAID相关知识

RAID0方案是无冗余和无校验的磁盘阵列，而RAID1-5方案均是加入了冗余（镜像）或校验的磁盘阵列。**能够提高 RAID 可靠性的措施主要是对磁盘进行镜像处理和奇偶校验**。

条带化技术是一种自动地将I/O的负载均衡到多个物理磁盘上的技术。条带化技术就是将一块连续的数据分成很多小部分并把它们分别存储到不同磁盘上去。这就能使多个进程同时访问数据的多个不同部分但不会造成磁盘冲突，**而且在需要对这种数据进行顺序访问的时候可以获得最大程度上的I/O并行能力，从而获得非常好的性能**。

### 中断I/O方式和DMA方式的比较

1. **中断处理方式**：
   + 在设备输入每个数据的过程中，由于无须 CPU 干预，因而**可使CPU与I/O设备并行工作**。仅当输完一个数据时，才需 CPU 花费极短的时间去做些中断处理，因此中断申请使用的是 CPU 处理时间；
   + 中断响应发生的时间是在**一条指令执行结束**之后；
   + 数据是在软件（**中断服务程序**）的控制下完成传送的。
2. **DMA方式**：
   + 数据传输的基本单位是**数据块**；
   + 每个数据块传送完毕时，都会发生**DMA请求**，DMA请求每次申请的是总线的使用权，所传送的数据是从设备直接送入内存的，或者相反；
   + 仅在传送一个或多个数据块的开始和结束时，才需 CPU 干预，整块数据的传送是在硬件（**DMA控制器**）的控制下完成的。

### IO密集型的进程和计算密集型的进程优先级

当系统中存在大量IO密集型任务时，**优先处理这类任务**可以减少IO等待时间，提高IO设备的利用率，同时可以充分利用CPU资源等待IO操作完成。

### *操作系统的加载*

![image-20231004142848998](https://picgo-myblog.oss-cn-beijing.aliyuncs.com/Images/image-20231004142848998.png)

操作系统最终被加载到RAM中。

### 预防死锁、避免死锁的区别

1. **预防死锁**：预防死锁的方法是在**设计阶段**就采取措施，使得系统不会发生死锁。这种方法主要通过破坏死锁产生的四个必要条件（互斥访问、不可剥夺、请求和保持、循环等待）来实现。**预防死锁是一种静态的、全局的策略**，需要在系统设计和实施阶段进行考虑和实施。
2. **避免死锁**：避免死锁是**在系统运行时根据进程的动态行为**来避免死锁的发生。这种方法会通过对资源的合理分配和释放（如银行家算法）来避免进入死锁状态。**避免死锁是一种动态的、局部的策略**，根据当前系统状态来进行判断和处理。

### 物理层基带传输信号

图源《通信原理》（樊昌信--第六版）

![image-20231004154036816](https://picgo-myblog.oss-cn-beijing.aliyuncs.com/Images/image-20231004154036816.png)

(a) 单极性非归零编码  
(b) 双极性非归零编码  
(c) 单极性归零编码  
(d) 双极性归零编码  
(e) 差分编码

图源王道《计算机网络考研复习指导》

![image-20231004154634765](https://picgo-myblog.oss-cn-beijing.aliyuncs.com/Images/image-20231004154634765.png)

以太网使用的编码方式就是曼彻斯特编码

### SMTP协议

![image-20231004163842082](https://picgo-myblog.oss-cn-beijing.aliyuncs.com/Images/image-20231004163842082.png)

![image-20231004163934927](https://picgo-myblog.oss-cn-beijing.aliyuncs.com/Images/image-20231004163934927.png)

## 14年

### 指令cache与数据cache分离的主要目的

可以减少指令流水线资源冲突。如一个进程在取指令时，另一条指令可以同时取数据。

### 管道通信的定义、特点

1. **管道是驻留在内存中的伪文件，可以使用标准文件函数对其进行访问。**与磁盘文件不同，它是临时的，且并不占用磁盘或者其他外部存储的空间，而是占用内存空间。Linux系统直接把管道实现成了一种文件系统，借助VFS给应用程序提供操作接口。**所以，Linux上的管道就是一个操作方式为文件的内存缓冲区。**
2. 管道的容量大小通常为内存上的一页，它的大小**并不是受磁盘容量大小的限制**。
3. 它常用于**父子**进程之间的通信。类似于通信中半双工信道的进程通信机制，一个管道同一个时刻只能最多有一个方向的传输，不能两个方向同时进行。若要实现父子进程双向通信，则需要定义两个管道。
4. 一个进程可以向管道写入数据，另一个进程可以同时从管道读取数据。当管道满时，进程在写管道会被阻塞，而当管道空时，进程在读管道会被阻塞。
5. 从管道读数据是一次性操作，数据一旦被读取，**就释放空间以便写更多数据**。

关于管道不是文件这一点，参考知乎文章[Linux 的进程间通信：管道](https://zhuanlan.zhihu.com/p/58489873)以及[Windows应用开发](https://learn.microsoft.com/zh-cn/windows/win32/ipc/interprocess-communications)

### 交换区和内存映射

内存映射和交换区（Swap）都是操作系统中用于管理内存的重要机制，但它们有着不同的作用和实现方式。

1. **内存映射**：

   + **作用**：内存映射是一种将磁盘上的文件或设备映射到进程的地址空间中的机制。它允许程序直接访问磁盘上的文件，而无需通过常规的文件读写操作。通过内存映射，程序可以将文件的内容视为内存中的一部分，从而实现对文件的高效访问。

   + **实现**：内存映射是通过将磁盘文件的某个区域映射到进程的虚拟地址空间中，从而让程序可以直接读写这块区域的内容。在某种程度上，内存映射可以看作是文件和内存之间的一个透明的接口。

2. **交换区**：

   + **作用**：交换区是一块硬盘空间，用于暂时存储在物理内存中暂时不活跃的数据和程序。当物理内存不足以容纳当前运行的程序和数据时，操作系统会将一部分数据移到交换区中，从而释放出物理内存。

   + **实现**：交换区是一块硬盘或者固态硬盘的空间，它用于作为物理内存的扩展。当系统需要释放物理内存时，它会将部分不活动的数据和程序写入到交换区中。当需要时，可以将交换区中的数据重新读取到物理内存中。

**区别**：

+ **作用不同**：内存映射主要用于实现文件和内存之间的高效访问，而交换区用于在物理内存不足时暂存数据。

+ **实现方式不同**：内存映射是通过将文件或设备映射到进程的地址空间中实现的，而交换区是通过将部分不活动的数据写入到硬盘空间中实现的。

+ **适用场景不同**：内存映射适用于需要频繁访问文件内容的场景，而交换区是用于解决物理内存不足的问题。

## 15年

### 卡特兰数

$\frac{1}{n+1}C^n_{2n}$

例如，n个不同的元素进栈，出栈元素不同排列的个数为$\frac{1}{n+1}C^n_{2n}$

例如，求先序序列为abcd的不同二叉树的个数也可以利用此解法：  
根据二叉树前序遍历和中序遍历的递归算法中递归工作栈的状态变化得出：前序序列和中序序列的关系相当于以前序序列为入栈次序，以中序序列为出栈次序。因为前序序列和中序序列可以唯一地确定一棵二叉树，所以题意相当于“以序列abcd为入栈次序，则出栈序列的个数为多少”，对于n个不同元素进栈，出栈序列的个数为卡特兰数。

### 堆调整时的比较次数

例如，对于大根堆：

1. **调整乱序初始堆**：从全部结点的一半处n/2，开始调整：
   + **调整i及其子树**：向下调整：**需要先两个孩子进行比较，再和较大者进行交换**；
   + **i--**：对左兄弟结点或父结点进行调整，重复上一步。
2. **插入节点后**：自底向上比较调整，每次向上调整，**只需要和父节点进行比较**，如果比父节点大，则上移，否则不动；而不是先和兄弟节点比较谁大，因为兄弟节点必定比父节点小。
3. **删除节点后**：自顶向下比较调整，每次上下调整，**需要先两个孩子进行比较，再和较大者进行交换**，因为本节点和两个孩子的大小关系、两个孩子之间的大小都未知。

### 浮点数溢出问题

1. 对阶操作不会引起阶码上溢或下溢（小阶向大阶对齐，这里前提是小阶能对齐到大阶，如果大的阶数超过了小阶数的阶码所能表示的范围，那么对阶时也会发生溢出）；
2. 左规时可能引起阶码下溢；
3. 右规和尾数舍入（0舍1入法）都可能引起阶码上溢；
4. **尾数溢出时，结果不一定溢出**，因为尾数溢出可以通过右规操作来纠正，结果可能产生误差，但不一定会溢出。

### DRAM的刷新

对于ROM，即使断电信息也不会丢失。

对于SRAM，其存储元（区别于存储单位）是双稳态触发器（六晶体管MOS），即使信息被读出后，**仍保持原状态而不需要再生**（非破坏性读出）。但是断电后信息会丢失。

对于DRAM，其存储元通常只使用一个晶体管。DRAM电容上的电荷一般只能维持1~2ms，**即使电源不断电，信息也会自动消失**，因此必须每隔一段时间进行刷新，称为刷新周期，常见的刷新方式有3种：  

1. **集中刷新**：指在一个刷新周期内，利用一段固定的时间，依次对存储器的所有行进行逐一再生，在此期间停止对存储器的读写操作，称为“死时间”，又称访存“死区”。优点是读写操作时不受刷新工作的影响；缺点是在集中刷新期间不能访问存储器。
2. 分散刷新：**把对每行的刷新分散到各个工作周期中**。这样，一个存储器的系统工作周期分为两部分：前半部分用于正常读、写或保持；后半部分用于刷新。优点是没有死区；缺点是加长了系统的存取周期。
3. 异步刷新：异步刷新是**前两种方法的结合**。具体做法是将刷新周期除以行数，得到两次刷新操作之间的时间间隔t，利用逻辑电路每隔时间t产生一次刷新请求。这样可以避免使CPU连续等待过长的时间，而且减少了刷新次数，从根本上提高了整机的工作效率。

DRAM 的刷新需要注意以下问题：

1. 刷新对CPU是透明的，即**刷新不依赖于外部的访问，不需要CPU控制**；
2. **DRAM的刷新单位是行**，由芯片内部自行生成行地址；
3. 刷新操作类似于读操作，但又有所不同。另外，**刷新时不需要选片，即整个存储器中的所有芯片同时被刷新**。

### *总线定时*

**同步定时方式：**系统采用一个统一的时钟信号来协调发送和接收双方的传送定时关系。  

**异步定时方式**：没有统一的时钟，也没有固定的时间间隔，完全依靠传送双方相互制约的“握手”信号来实现定时控制。

1. 不互锁方式：主设备发出“请求”信号后，不必等到接到从设备的“回答”信号，而是经过一段时间，便撤销“请求”信号。而从设备在接到“请求”信号后，发出“回答”信号，并经过一段时间，自动撤销“回答”信号。**双方不存在互锁关系**。**速度最快，可靠性最差**
2. 半互锁方式：主设备发出“请求”信号后，必须待接到从设备的“回答”信号后，才撤销“请求”信号，**有互锁的关系**。而从设备在接到“请求”信号后，发出“回答”信号，但不必等待获知主设备的“请求”信号已经撤销，而是隔一段时间后自动撤销“回答”信号，**不存在互锁关系**；
3. 全互锁方式：主设备发出“请求”信号后，必须待从设备“回答”后，才撤销“请求”信号；从设备发出“回答”信号，必须待获知主设备“请求”信号已撤销后，再撤销其“回答”信号。**双方存在互锁关系**。**速度最慢，可靠性最好**

优点：总线周期长度可变，能保证两个工作速度相差很大的部件或设备之间可靠地进行信息交换，自动适应时间的配合。
缺点：比同步控制方式稍复杂一些，速度比同步定时方式慢

**半同步定时方式**：**统一时钟的基础上**，增加一个“等待”响应信号；

**分离式通信**：$\cdots$

### DHCP报文地址问题

![image-20231009210122937](https://picgo-myblog.oss-cn-beijing.aliyuncs.com/Images/image-20231009210122937.png)
