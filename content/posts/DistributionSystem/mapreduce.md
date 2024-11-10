---
title: "[论文翻译]MapReduce: Simplified Data Processing on Large Clusters"
date: 2024-11-06
draft: false
tags: ["分布式", "论文"]
categories: ["Learning"]
---

## 摘要

MapReduce 是一种用于处理和生成大型数据集的编程模型和相关实现。用户指定一个 **map** 函数来处理键/值对，生成一组中间键/值对，并指定一个 **reduce** 函数来合并与同一中间键相关的所有中间值。如本文所示，许多现实世界中的任务都可以用这个模型来表达。

以这种函数式风格编写的程序会自动并行化，并在大型商用机器集群上执行。运行时系统会处理分割输入数据、调度程序在一组机器上的执行、处理机器故障以及管理所需的机器间通信等细节。这样，没有任何并行和分布式系统经验的程序员也能轻松利用大型分布式系统的资源。

我们的 MapReduce 实现运行在大型商用机器集群上，并且具有高度可扩展性：典型的 MapReduce 计算在数千台机器上处理数 TB 的数据。程序员发现该系统易于使用：数百个 MapReduce 程序已经实现，并且每天在 Google 集群上执行超过一千个 MapReduce 作业。

## 1 引言

在过去五年中，本文作者和 Google 的许多其他人已经实现了数百种特殊用途的计算，这些计算处理大量原始数据（例如抓取的文档、Web 请求日志等），以计算各种派生数据（例如倒排索引、Web 文档图形结构的各种表示、每个主机抓取的页面数量摘要、给定一天中最频繁的查询集等）。大多数此类计算在概念上都很简单。但是，输入数据通常很大，并且计算必须分布在数百或数千台机器上才能在合理的时间内完成。如何并行化计算、分发数据和处理故障的问题共同掩盖了原始的简单计算，并使用大量复杂的代码来处理这些问题。

为了应对这种复杂性，我们设计了一种新的抽象，它允许我们表达我们试图执行的简单计算，但隐藏了库中的并行化、容错、数据分布和负载平衡的复杂细节。我们的抽象受到 Lisp 和许多其他函数式语言中存在的 map 和 Reduce 原语的启发。我们意识到，我们的大多数计算都涉及将 map 操作应用于输入中的每个逻辑“记录”，以计算一组中间键/值对，然后对共享相同键的所有值应用 Reduce 操作，以便适当地组合派生数据。我们使用具有用户指定的 map 和 Reduce 操作的函数模型，使我们能够轻松地并行化大型计算，并使用重新执行作为容错的主要机制。

这项工作的主要贡献在于提供了一个简单而功能强大的接口，可实现大规模计算的自动并行化和分布式处理，同时还实现了该接口在大型商用 PC 集群上的高性能。

第 2 节描述了基本编程模型并给出了几个示例。第 3 节描述了针对我们基于集群的计算环境定制的 MapReduce 接口实现。第 4 节描述了我们发现有用的编程模型的几个改进。第 5 节对我们的实现进行了各种任务的性能测量。第 6 节探讨了 MapReduce 在 Google 中的使用情况，包括我们使用它作为重写生产索引系统的基础的经验。第 7 节讨论了相关工作和未来工作。

## 2 编程模型

计算接收一组**输入**键/值对，并产生一组**输出**键/值对。MapReduce 库的用户将计算表述为两个函数：**Map** 和 **Reduce**。

Map 由用户编写，它接受一个输入对并生成一组中间键/值对。MapReduce 库将与同一中间键 I 相关的所有中间值分组在一起，并将它们传递给 Reduce 函数。

### 2.1 示例

考虑一下计算大量文件集中每个单词出现次数的问题。用户可以编写类似于以下伪代码的代码：

```c++
map(String key, String value):
    // key: document name
    // value: document content
    for each word w in value:
        EmitIntermediate(w, "1");

reduce(String key, Iterator values):
    // key: a word
    // values: a list of counts
    int result = 0;
    for each v in values:
        result += ParseInt(v);
    Emit(AsString(result));
```

`map` 函数输出每个单词以及相关的出现次数（在这个简单的例子中只有"1"）。`reduce` 函数将针对特定单词发出的所有计数相加。

此外，用户编写代码以将输入和输出文件的名称以及可选的调整参数填充到 **mapreduce 规范对象**中。然后，用户调用 MapReduce 函数，将规范对象传递给它。用户的代码与 MapReduce 库（以 C++ 实现）链接在一起。附录 A 包含此示例的完整程序文本。

### 2.2 类型

尽管前面的伪代码是用字符串输入和输出编写的，但从概念上讲，用户提供的 map 和 Reduce 函数具有相关类型：

```c++
map (k1,v1) → list(k2,v2)  

reduce (k2,list(v2)) → list(v2)
```

也就是说，输入键值与输出键值来自不同的域。此外，中间键值与输出键值来自同一域。

我们的 C++ 实现将字符串传入和传出用户定义的函数，并让用户代码在字符串和适当类型之间进行转换。

### 2.3 更多例子

下面是几个有趣程序的简单示例，它们可以轻松地表示为 MapReduce 计算。

**分布式 Grep**：map 函数会在与所提供的模式匹配时输出一行数据。reduce 函数是一个 identify 函数，它只是将提供的中间数据复制到输出中。

**URL 访问频率计数**：map 函数处理网页请求日志并输出`<URL, 1>`。reduce 函数将同一 URL 的所有值相加，输出一对 `<URL, total count>`。

**反向 Web 链接图**：map 函数针对在名为 `source` 的页面中找到的目标 URL 的每个链接输出 `<target, source>` 对。reduce 函数连接与给定目标 URL 关联的所有源 URL 列表并发出以下对：`<target, list(source)>`.

**每主机的术语向量**：术语向量将出现在文档或文档集中的最重要单词总结为`<word, f requency>`对的列表。map函数为每个输入文档发出`<hostname, term vector>`对（主机名是从文档的 URL 中提取的）。reduce 函数接收给定主机的所有文档术语向量。它将这些术语向量相加，舍弃不常见的术语，然后发出最终的`<hostname, term vector>`对。

**反向索引**：map函数解析每个文档，并输出`<word, document ID>`对的序列。reduce 函数接受给定词的所有词对，对相应的文档 ID 进行排序，然后输出`<word, list(document ID)>`词对。所有输出词对的集合构成了一个简单的倒排索引。这种计算方法很容易扩展，以跟踪单词的位置。

**分布式排序**：map 函数从每条记录中提取键，并输出`<key, record>`对。reduce函数会原封不动地输出所有记录对。这种计算依赖于第 4.1 节中描述的分区设施和第 4.2 节中描述的排序属性。

## 3 实现

MapReduce 接口有许多不同的实现方式。正确的选择取决于环境。例如，一种实现可能适用于小型共享内存机器，另一种可能适用于大型 NUMA 多处理器，还有一种可能适用于更大规模的联网机器。

本节介绍针对谷歌广泛使用的计算环境的实施方案：通过交换式以太网连接在一起的大型商用 PC 集群 [^4]。在我们的环境中：

(1) 机器通常是运行 Linux 的双处理器 x86 处理器，每台机器有 2-4 GB 内存。

(2) 使用普通网络硬件--在机器层面通常为 100 Mb/s或 1 Gb/s，但平均整体带宽要低得多。

(3) 集群由成百上千台机器组成，因此机器故障很常见。

(4) 存储由直接连接到各个机器的廉价 IDE 磁盘提供。内部开发的分布式文件系统 [^8] 用于管理存储在这些磁盘上的数据。文件系统使用复制来在不可靠的硬件上提供可用性和可靠性。

(5) 用户向调度系统提交作业。每个作业由一组任务组成，并由调度程序映射到集群内的一组可用机器上。

## 3.1 执行概述

Map 调用通过自动将输入数据划分为一组 M 个分片，分布在多台机器上。输入分片可以由不同的机器并行处理。Reduce 调用通过使用分区函数（例如 `hash(key) mod R`）将中间键空间划分为 R 个部分来分布。分区数 (R) 和分区函数由用户指定。

![Figure 1: Execution overview](https://picgo-01.oss-cn-shanghai.aliyuncs.com/Images/mapreduceFigure1.png "Figure 1: Execution overview")

**图 1** 显示了我们实现 MapReduce 操作的整体流程。当用户程序调用 MapReduce 函数时，会发生以下一系列操作（图 1 中的编号标签与下面列表中的编号相对应）：

1. 用户程序中的 MapReduce 库首先会将输入文件分割成 M 个片段，每个片段通常为 16 MB 至 64 MB（用户可通过可选参数进行控制）。然后，它在机器集群上启动程序的多个副本。
2. 程序中的一个副本是特殊的--master。其余的是由master分配工作的worker。有 M 个map任务和 R 个reduce任务需要分配。master 会挑选空闲的 worker，为每个worker分配一个map任务或reduce任务。
3. 分配给 Map 任务的 worker 会读取相应输入分片的内容。它从输入数据中解析出键/值对，并将每个键/值对传递给用户定义的 Map 函数。Map 函数生成的中间键/值对在内存中缓冲。
4. 缓冲数据对会定期写入本地磁盘，并由分区函数分割成 R 个区域。本地磁盘上这些缓冲对的位置会传回master，由master负责将这些位置转发给执行 reduce 的worker。
5. 当 master 通知执行 reduce 的worker 这些位置时，它使用远程过程调用从执行 map的worker的本地磁盘读取缓冲数据。当执行 reduce 的worker读取了所有中间数据时，它会按中间键对其进行排序，以便将同一键的所有匹配项组合在一起。排序是必需的，因为通常许多不同的 key 映射到同一个 reduce 任务。如果中间数据量太大而无法放入内存，则使用外部排序。
6. reduce 工作程序迭代排序的中间数据，对于遇到的每个唯一中间键，它将键和相应的中间值集传递给用户的 Reduce 函数。Reduce 函数的输出将附加到此 reduce 分区的最终输出文件中。
7. 当 map 任务和 reduce 任务全部完成后，master 会唤醒用户程序。 此时，用户程序中的 MapReduce 调用返回给用户代码。

成功完成后，mapreduce 执行的输出可在 R 输出文件中获得（每个 Reduce 任务一个，文件名由用户指定）。通常，用户不需要将这些 R 输出文件合并为一个文件 - 他们经常将这些文件作为输入传递给另一个 MapReduce 调用，或者从能够处理分成多个文件的输入的另一个分布式应用程序中使用它们。

### 3.2 Master 数据结构

Master 保存了几个数据结构。对于每个 Map 任务和 Reduce 任务，它存储了状态（空闲、进行中或已完成）和 Worker 机器的标识（对于非空闲任务）。

master是将中间文件区域的位置从 Map 任务传播到 Reduce 任务的管道。因此，对于每个已完成的 Map 任务，master都会存储 Map 任务生成的 R 个中间文件区域的位置和大小。Map 任务完成后，将收到对此位置和大小信息的更新。这些信息将逐步推送到具有正在进行的 Reduce 任务的worker。

### 3.3 容错

由于 MapReduce 库旨在帮助使用数百或数千台机器处理大量数据，因此该库必须能够优雅地容忍机器故障。

#### Worker 失败

Master 定期对每个 Worker 执行 ping 操作。如果在一定时间内没有收到 Worker 的响应，Master 会将该 Worker 标记为失败。worker 完成的任何map任务都会重置回其初始空闲状态，因此有资格在其他worker 上进行调度。同样，发生故障的worker 上正在进行的任何map任务或reduce任务也会重置为空闲状态，并有资格重新安排。

已完成的map任务会在发生故障时重新执行，因为它们的输出存储在故障计算机的本地磁盘上，因此无法访问。已完成的reduce任务不需要重新执行，因为它们的输出存储在全局文件系统中。

当一个map任务先由worker A执行，然后由worker B执行时（因为A失败），所有执行reduce任务的worker都会收到重新执行的通知。任何尚未从worker A读取数据的reduce任务将从worker B读取数据。

MapReduce 对大规模工作故障具有弹性。  例如，在一次 MapReduce 操作期间，正在运行的集群上的网络维护导致一次 80 台计算机组在几分钟内无法访问。 MapReduce master只是重新执行了无法访问的worker机器完成的工作，并继续向前推进，最终完成MapReduce操作。

#### Master 失败

很容易让 master 写入上述master数据结构的周期性检查点。如果master任务终止，可以从最后一个检查点状态开始一个新的副本。然而，考虑到只有一个master，它发生故障的可能性不大；因此，如果master发生故障，我们当前的实现将中止 MapReduce 计算。客户端可以检查此情况并根据需要重试 MapReduce 操作。

#### 出现故障时的语义

当用户提供的 map 和 reduce 操作能够根据输入值确定性地计算出结果时，我们的分布式实现将产生与程序在没有故障的情况下顺序执行时相同的输出。

我们通过原子性地提交 map 和 reduce 任务的输出来确保这一特性。每个正在执行的任务都会将其输出写入临时文件。一个 reduce 任务生成一个这样的文件，而一个 map 任务则为每个 reduce 任务生成一个文件，共 R 个。当一个 map 任务完成后，worker 会向 master 发送一条消息，并附上这 R 个临时文件的名称。如果 master 收到的是一个已经完成的 map 任务的完成消息，它会忽略这条消息。否则，它会在master的数据结构中记录这 R 个文件的名称。

当一个 reduce 任务完成后，负责该任务的 worker 会将临时输出文件原子性地重命名为最终输出文件。如果这个 reduce 任务在多台机器上同时执行，那么会有多次重命名操作指向同一个最终输出文件。我们依靠文件系统提供的原子性重命名功能来确保最终文件系统中只包含一次 reduce 任务执行所生成的数据。

我们的 map 和 reduce 操作大多数都是确定性的，这种情况下，我们的语义等同于顺序执行，这让程序员能够更容易地理解程序的运行行为。当 map 和/或 reduce 操作是非确定性的时候，我们提供了一种较弱但仍然合理的语义。在非确定性操作符的情况下，特定 reduce 任务 $R_1$ 的输出与该非确定性程序顺序执行产生的 $R_1$ 输出相同。但是，对于另一个 reduce 任务 $R_2$，其输出可能来自于该非确定性程序的不同顺序执行所产生的 $R_2$ 输出。

考虑 map 任务 M 和 reduce 任务 $R_1$ 和 $R_2$。设 $e(R_i)$ 为已提交的 Ri 执行实例（每个 $R_i$ 只有一个这样的执行实例）。较弱的语义之所以出现，是因为 e($R_1$) 可能读取了 M 任务某次执行产生的输出，而 e($R_2$) 可能读取了 M 任务另一次执行产生的输出。

### 3.4 位置

在我们的计算环境中，网络带宽是一种相对宝贵的资源。我们通过利用输入数据（由 GFS [^8] 管理）已存储在集群中机器的本地磁盘上的优势，来减少网络带宽的使用。GFS 将每个文件划分为 64 MB 的数据块，并在不同的机器上存储每个块的多个副本（通常是 3 个）。MapReduce master会考虑输入文件的位置信息，并尽可能在一个拥有相应输入数据副本的机器上安排 map 任务。如果不可能，它会尝试在靠近该任务输入数据副本的地方安排 map 任务（例如，在与数据所在机器相同的网络交换机上的 worker 机器上）。当在集群中大量 worker 上运行大型 MapReduce 操作时，大部分输入数据都能在本地读取，从而不占用网络带宽。

### 3.5 任务粒度

我们按照上述方法将 map 阶段划分为 M 个小部分，将 reduce 阶段划分为 R 个小部分。理想情况下，M 和 R 应该远大于工作机器的数量。让每台工作机器执行多个不同的任务有助于提升动态负载均衡，同时在工作机器出现故障时也能加快恢复速度：它已完成的众多 map 任务可以分配到其他所有工作机器上去执行。

在我们的实现中，M 和 R 的数值实际上存在一定的实际限制。这是因为master需要做出 O(M + R) 次调度决策，并且需要在内存中保持 O(M ∗ R) 的状态，如前所述。（不过，内存使用的常数因子相对较小：O(M ∗ R) 部分的状态大约由每个 map 任务/reduce 任务对的一个字节数据组成。）

此外，R 的值通常受到用户的限制，因为每个 reduce 任务的输出会分别存储在不同的输出文件中。在实际操作中，我们通常设定 M 的值，使得每个任务处理的输入数据大约在 16 MB 到 64 MB 之间（这样可以使上述的局部性优化效果最大化），而 R 则设定为预期使用的工作机器数量的一小倍数。在我们的计算中，常常使用 M = 200,000 和 R = 5,000，并部署 2,000 台工作机器进行 MapReduce 计算。

### 3.6 后备任务

MapReduce 操作总耗时增加的一个常见原因是“慢速机器”，也就是在计算的最后阶段，某台机器完成 map 或 reduce 任务所需的时间异常长。慢速机器的出现可能由多种因素引起。例如，一个磁盘出现问题的机器可能会频繁遇到可纠正的错误，这会使其读取性能从每秒 30 MB 降至每秒 1 MB。集群调度系统可能在同一台机器上安排了其他任务，由于 CPU、内存、本地磁盘或网络带宽的竞争，导致 MapReduce 代码执行速度减慢。我们最近遇到的一个问题是机器初始化代码中的一个错误，导致处理器的缓存功能被禁用，这使受影响的机器上的计算速度降低了超过一百倍。

我们实现了一个通用机制来减轻慢速机器带来的问题。当一个 MapReduce 操作即将完成时，master会安排剩余未完成任务的后备执行。无论原任务还是后备任务先完成，该任务即被视为完成。我们已经对这个机制进行了优化，使得它通常只会使操作使用的计算资源增加几个百分点。我们发现，这种方法显著缩短了完成大型 MapReduce 操作所需的时间。例如，当禁用后备任务机制时，第 5.3 节中描述的排序程序完成时间会延长 44%。

## 4 改进

虽然简单编写 Map 和 Reduce 函数提供的基本功能足以满足大多数需求，但我们发现一些有用的扩展。本节对此进行了描述。

### 4.1 分区函数

MapReduce 用户可以指定所需的 reduce 任务/输出文件数量 R。数据通过中间键的分区函数分配到这些任务中。系统提供了一个默认的分区函数，它使用哈希（例如，“`hash(key)` mod R”），这通常能产生相当均衡的数据分区。但在某些情况下，根据键的其他属性来分区数据会更有用。例如，如果输出键是 URL，我们可能希望同一个主机的所有记录都保存在同一个输出文件中。为了应对这类需求，MapReduce 库的用户可以提供一个自定义的分区函数。例如，使用“`hash(Hostname(urlkey))` mod R”作为分区函数，就能确保同一个主机的所有 URL 都保存在同一个输出文件中。

### 4.2 顺序保证

我们确保在每个数据分区内部，中间键/值对按照键的递增顺序进行处理。这样的顺序保证使得为每个分区生成一个有序的输出文件变得简单，这在需要支持通过键进行高效随机访问查找的输出文件格式中非常有用，或者当输出文件的最终用户发现数据有序排列更方便时也很实用。

### 4.3 合并函数

在某些情况下，每个 map 任务产生的中间键可能会有很多重复，而且用户定义的 Reduce 函数具有可交换和可结合的特性。一个典型的例子是第 2.1 节中的单词计数案例。由于单词出现的频率通常遵循 Zipf 分布，每个 map 任务可能会生成数百或数千条 <the, 1> 这种格式的记录。所有这些计数结果都会被发送到同一个 reduce 任务，并由 Reduce 函数合并成一个总数。为了优化这个过程，我们允许用户指定一个可选的 Combiner 函数，这个函数可以在数据发送到网络之前对数据进行局部合并。

Combiner 函数在执行 map 任务的每台机器上运行。通常，相同的代码被用于实现 combiner 函数和 reduce 函数。reduce 函数与 combiner 函数之间的唯一区别在于 MapReduce 库处理函数输出的方式不同。reduce 函数的输出会直接写入最终的输出文件，而 combiner 函数的输出则被写入一个中间文件，该文件随后会被发送到 reduce 任务进行处理。

局部合并大大加快了一些特定类型的 MapReduce 操作。附录 A 中提供了一个使用 combiner 函数的示例。

### 4.4 输入与输出类型

MapReduce 库提供了支持多种不同格式读取输入数据的功能。例如，在“文本”模式输入中，每行被视为一个键/值对：键是文件中的位置偏移，值是行的内容。另一种常见且受支持的格式是按键排序的键/值对序列。每种输入类型的实现都了解如何将自己分割成有意义的范围，以便作为独立的 map 任务进行处理（例如，文本模式的范围分割确保只在行边界进行分割）。用户可以通过实现一个简单的读取器接口来添加对新输入类型的支持，尽管大多数用户通常只使用少数几个预定义的输入类型之一。

读取器并不一定非要从文件中读取数据。例如，可以很容易地定义一个从数据库中读取记录的读取器，或者从内存中映射的数据结构中读取记录。

同样地，我们支持一系列输出类型，用于以不同格式生成数据，用户编写的代码可以轻松地添加对新输出类型的支持。

### 4.5 副作用

在某些情况下，MapReduce 的用户发现从他们的 map 和/或 reduce 操作中生成辅助文件作为额外的输出非常方便。我们依赖于应用程序编写者确保这样的副作用操作具有原子性和幂等性。通常，应用程序会写入一个临时文件，并在文件完全生成后原子性地重命名这个文件。我们不支持单个任务生成的多个输出文件的原子性两阶段提交。因此，产生多个输出文件且需要跨文件一致性的任务应该是确定性的。在实际应用中，这个限制从未造成过问题。

### 4.6 跳过坏记录

有时用户编写的代码中可能存在导致 Map 或 Reduce 函数在处理特定记录时始终崩溃的 bug。这类 bug 会阻止 MapReduce 操作的完成。通常的解决方法当然是修复这些 bug，但有时这并不可行，比如 bug 出现在一个没有提供源代码的第三方库中。有时候，忽略一些记录是可以接受的，例如在进行大量数据集的统计分析时。为了应对这种情况，我们提供了一个可选的执行模式，MapReduce 库能够检测出哪些记录会导致崩溃，并跳过这些记录，以便继续执行操作。

每个工作进程都设置了一个信号处理程序，用于捕获段错误和总线错误。在调用用户的 Map 或 Reduce 操作之前，MapReduce 库会将参数的序列号保存在一个全局变量中。如果用户代码触发了信号，信号处理程序会发送一个包含该序列号的“最后努力”UDP 数据包给 MapReduce master。当master发现某个特定记录上出现多次失败时，它会在下一次重新调度相应的 Map 或 Reduce 任务时指示应该跳过这个记录。

### 4.7 本地执行

调试 Map 或 Reduce 函数中的问题可能相当复杂，因为实际计算是在一个分布式系统中进行的，通常涉及数千台机器，而工作分配的决策是由master动态做出的。为了帮助简化调试、性能分析和进行小规模测试，我们开发了一个 MapReduce 库的替代版本，它可以在本地机器上顺序执行 MapReduce 操作的所有工作。这个版本为用户提供了控制选项，可以将计算限制在特定的 map 任务上。用户只需在运行程序时添加一个特殊标志，就可以轻松地使用任何他们觉得有用的调试或测试工具（比如 gdb）。

### 4.8 状态信息

master运行一个内部 HTTP 服务器，并提供了一系列供人查看的状态页面。这些页面显示了计算的进展情况，例如完成了多少任务，正在进行中的任务数量，输入数据、中间数据和输出数据的字节数，以及处理速率等信息。页面还提供了每个任务生成的标准错误和标准输出文件的链接。用户可以利用这些数据来预测计算所需的时间，并决定是否需要为计算添加更多资源。这些页面还可以帮助确定计算是否远低于预期速度。

此外，主页面还显示了哪些工作机器出现了故障，以及它们在故障时正在处理的 map 和 reduce 任务。这些信息对于诊断用户代码中的错误非常有用。

### 4.9 计数器

MapReduce 库提供了一个计数器功能，用于记录各种事件的发生次数。例如，用户代码可能需要统计处理过的总词数或索引的德语文档数量等。

要使用这个功能，用户代码需要创建一个具有名称的计数器对象，然后在 Map 和/或 Reduce 函数中根据需要增加计数器的值。例如：

```c++
Counter* uppercase; 
uppercase = GetCounter("uppercase");  
map(String name, String contents): 
  for each word w in contents: 
    if (IsCapitalized(w)): 
      uppercase->Increment(); 
    EmitIntermediate(w, "1");
```

各个工作机器上的计数器值会定期发送到master（随 ping 响应一起发送）。master会汇总成功完成的 map 和 reduce 任务的计数器值，并在 MapReduce 操作完成后返回给用户代码。当前的计数器值也会显示在master的状态页面上，以便人们可以监控计算的实时进展。在汇总计数器值时，master会排除同一 map 或 reduce 任务重复执行的影响，以避免重复计算。（重复执行可能是由我们使用后备任务或因故障而重新执行任务所导致的。）

MapReduce 库自动记录一些计数器值，如处理的输入键/值对数量和生成的输出键/值对数量。

用户发现计数器功能对于验证 MapReduce 操作的正确性非常有帮助。例如，在某些 MapReduce 操作中，用户代码可能需要确保生成的输出对数量与处理的输入对数量完全相等，或者处理的德语文档数量占所有处理文档数量的比例在可接受的范围内。

## 5 性能

在这节中，我们评估了 MapReduce 在一个大型计算机集群上执行两个计算任务的表现。其中一个任务是在大约一兆字节的数据中寻找特定模式。另一个任务是对大约一兆字节的数据进行排序。

这两个程序是用户编写的 MapReduce 程序中的典型代表。一类程序负责将数据在不同的表示形式之间进行转换，另一类程序则从大量数据中筛选出有价值的信息。

### 5.1 集群配置

所有程序都在一个包含大约 1800 台机器的集群上运行。每台机器都安装有两个 2GHz 的 Intel Xeon 处理器并开启了超线程功能，拥有 4GB 内存，两个 160GB 的 IDE 硬盘，以及千兆以太网连接。这些机器构成了一个两层树状交换网络，其根部的总带宽约为 100-200 Gbps。由于所有机器位于同一托管设施内，任意两台机器之间的往返延迟不到一毫秒。

在 4GB 内存中，大约有 1-1.5GB 被集群上运行的其他任务占用。程序是在一个周末下午执行的，那时 CPU、硬盘和网络的使用率相对较低。

### 5.2 Grep

**grep** 程序检查了 $10^{10}$ 个 100 字节长的记录，寻找一个出现频率较低的三字符模式（该模式在 92337 个记录中出现）。输入数据被划分为大约 64MB 的片段（M = 15000），所有输出结果被保存在一个文件中（R = 1）。

![Figure 2: Data transfer rate over time](https://picgo-01.oss-cn-shanghai.aliyuncs.com/Images/mapreduceFigure2.png "图 2：随时间变化的数据传输速率")

**图 2** 展示了计算过程随时间的变化。Y 轴表示输入数据的扫描速率。随着更多机器加入到这个 MapReduce 任务中，扫描速率逐渐加快，当分配了 1764 个工作节点时，速率达到峰值，超过 30 GB/s。随着 map 任务的完成，速率开始下降，在大约 80 秒后降至零。整个计算过程大约需要 150 秒，包括大约一分钟的启动时间。启动时间包括了将程序分发到所有工作节点以及与 GFS 交互，打开 1000 个输入文件并获取本地优化所需信息的时间延迟。

### 5.3 Sort

![Figure 3: Data transfer rates over time for different executions of the sort program](https://picgo-01.oss-cn-shanghai.aliyuncs.com/Images/mapreduceFigure3.png "图 3：排序程序的不同执行随时间变化的数据传输速率")

sort 程序对 $10^{10}$ 个 100 字节的记录（大约 1 TB 的数据）进行排序。该程序以 TeraSort 基准测试 [^10] 为蓝本。

排序程序的用户代码部分少于 50 行。一个三行的 Map 函数负责从文本行中提取一个 10 字节的排序键，并将这个键和原始文本行作为中间键/值对输出。我们使用了一个内置的 Identity 函数作为 Reduce 操作符，这个函数会将中间键/值对原封不动地作为输出键/值对。排序后的最终输出被写入一组两路复制的 GFS 文件中（也就是说，程序的输出是 2 兆字节的数据）。

和之前一样，输入数据被划分为 64MB 的片段（M = 15000）。我们将排序后的结果分成 4000 个文件（R = 4000）。分区函数通过检查键的起始字节来确定每个键应该分配到哪一部分。

在这个基准测试中，我们的分区函数已经预先知道了键的分布情况。在一般的排序程序中，我们会先进行一次 MapReduce 操作来收集键的样本，然后根据这些样本键的分布情况来计算最终排序阶段所需的分割点。

图 3 (a) 展示了排序程序正常执行的过程。左上角的图表显示了输入数据的读取速度。这个速度在达到大约 13 GB/s 后达到顶峰，然后迅速下降，因为所有的 map 任务在大约 200 秒内完成了。请注意，这个读取速度比 grep 程序的要慢。这是因为排序任务的 map 阶段大约有一半的时间和 I/O 带宽用于将中间结果写入本地磁盘，而 grep 程序的中间结果大小几乎可以忽略不计。

左中角的图表显示了数据从 map 任务传输到 reduce 任务的速率。这个过程在第一个 map 任务完成后立即开始。图表中的第一个高峰对应于大约 1700 个 reduce 任务的第一批（整个 MapReduce 任务被分配了大约 1700 台机器，每台机器同时最多执行一个 reduce 任务）。在大约 300 秒的计算过程中，一些第一批的 reduce 任务完成了，我们开始为剩下的 reduce 任务传输数据。所有的数据传输在大约 600 秒的计算过程中完成。

左下角的图表展示了 reduce 任务将排序后的数据写入最终输出文件的速度。在第一次数据洗牌结束和写入开始之间有一个时间差，因为机器正在对中间数据进行排序。写入操作以大约 2-4 GB/s 的速度持续了一段时间。所有的写入在大约 850 秒的计算过程中完成。包括启动时间在内，整个计算过程耗时 891 秒，这与目前 TeraSort 基准测试报告[^18]的最佳结果 1057 秒相近。

需要关注的几个点是：由于我们的本地优化策略，输入速率高于数据洗牌速率和输出速率——大部分数据是从本地磁盘读取的，绕过了我们相对受限的网络带宽。数据洗牌速率高于输出速率，因为输出阶段需要写入排序数据的两个副本（我们为了确保数据的可靠性和可用性，制作了输出的两个副本）。我们之所以写入两个副本，是因为这是我们底层文件系统提供的确保数据可靠性和可用性的机制。如果底层文件系统使用的是擦除编码 [^14] 而非复制，那么写入数据的网络带宽需求将会降低。

### 5.4 备份任务的效果

在图 3 (b) 中，我们展示了当禁用备份任务时排序程序的执行情况。整体执行流程与图 3 (a) 相似，但存在一个明显的长尾阶段，期间几乎没有任何写入活动。在经过 960 秒后，除了 5 个 reduce 任务外，其余所有任务都已结束。然而，这几个剩余的慢任务又花了额外的 300 秒才最终完成。整个计算过程耗时 1283 秒，相比正常执行时间增加了 44%。

### 5.5 机器故障

在图 3 (c) 中，我们展示了在计算进行了几分钟之后，故意杀死了 1746 个工作进程中的 200 个的排序程序执行情况。由于只有进程被终止，而机器本身仍然正常工作，底层的集群调度程序迅速在这些机器上重新启动了新的工作进程。

工作进程的终止在图表上显示为负的输入速率，因为一些已经完成的 map 任务消失（由于相应的 map 工作进程被终止），需要重新执行。这些 map 任务的重新执行相对迅速。整个计算过程包括启动开销在内耗时 933 秒，仅比正常执行时间增加了 5%。

## 6 经验

![Figure 4: MapReduce instances over time](https://picgo-01.oss-cn-shanghai.aliyuncs.com/Images/mapreduceFigure4.png "图 4：MapReduce 实例随时间的变化")

![Table 1: MapReduce jobs run in August 2004](https://picgo-01.oss-cn-shanghai.aliyuncs.com/Images/mapreduceTable1.png "表 1：2004 年 8 月运行的 MapReduce 作业")

我们在 2003 年 2 月开发了 MapReduce 库的第一个版本，并在同年 8 月对其进行了重大升级，增加了本地优化、跨工作计算机任务执行的动态负载平衡等。自那时起，我们惊喜地发现 MapReduce 库在我们处理的各种问题中有着广泛的应用。它已经被 Google 内部的多个领域所采用，包括：

+ 大规模机器学习问题
+ Google 新闻和 Froogle 产品的聚类问题
+ 提取用于生成热门查询报告的数据（例如 Google Zeitgeist）
+ 为新实验和产品提取网页的属性（例如，从大量网页语料库中提取地理位置以进行本地化搜索），以及
+ 大规模图计算

图 4 展示了在我们的主要源代码管理系统中，独立 MapReduce 程序的数量随时间急剧增加，从 2003 年初的 0 个增加到 2004 年 9 月底的近 900 个实例。MapReduce 取得巨大成功的原因是它允许开发者编写简单的程序，并在半小时内在成千上万台机器上高效运行，这极大地加快了开发和原型设计的速度。此外，它还让那些没有分布式或并行系统经验的程序员能够轻松地利用大量资源。

每个作业完成后，MapReduce 库会记录下作业使用的计算资源统计数据。在表 1 中，我们展示了 2004 年 8 月在 Google 运行的一些 MapReduce 作业的统计信息。

### 6.1 大规模索引

到目前为止，我们对 MapReduce 最重要的用途之一是完全重写了生产索引系统，该系统生成用于 Google 网络搜索服务的数据结构。索引系统将我们的爬虫系统检索到的大量文档作为输入，这些文档存储为一组 GFS 文件。这些文档的原始内容超过 20 TB 的数据。索引过程以五到十个 MapReduce 操作的序列运行。使用 MapReduce（而不是索引系统先前版本中的临时分布式传递）提供了几个好处：

+ 索引代码更简单、更小、更易于理解，因为处理容错、分布和并行化的代码隐藏在 MapReduce 库中。例如，使用 MapReduce 表达时，一个计算阶段的大小从大约 3800 行 C++ 代码下降到大约 700 行。
+ MapReduce 库的性能足够好，因此我们可以将概念上不相关的计算分开，而不是将它们混合在一起，以避免对数据进行额外的传递。这使得更改索引过程变得很容易。例如，在旧索引系统中需要几个月才能完成的一项更改在新系统中仅用了几天就完成了。
+ 索引过程变得更容易操作，因为大多数由机器故障、机器速度慢和网络故障引起的问题都由 MapReduce 库自动处理，无需操作员干预。此外，通过向索引集群添加新机器，可以轻松提高索引过程的性能。

## 7 相关工作

许多系统通过限制编程模型来自动实现计算的并行化。例如，使用并行前缀计算[^6][^9][^13]，可以在 N 个处理器上以 O(log N) 的时间复杂度计算一个 N 元素数组的所有前缀的关联函数。MapReduce 可以看作是基于我们在大型实际计算中的经验，对这些模型进行简化和提炼的结果。更重要的是，我们提供了一个能够扩展到数千个处理器上的容错实现。与此相反，大多数并行处理系统只在较小的规模上实现，并且将处理机器故障的细节留给程序员去处理。

BSP（Bulk Synchronous Programming）[^17] 和一些 MPI（Message Passing Interface）原语 [^11] 提供了更高级别的抽象，使程序员更容易编写并行程序。这些系统与 MapReduce 的关键区别在于，MapReduce 通过限制编程模型来自动并行化用户程序，并提供了透明的容错能力。

我们的本地优化受到了主动磁盘 [^12][^15] 等技术的启发，这些技术将计算推向靠近本地磁盘的处理元素，以减少跨 I/O 子系统或网络发送的数据量。我们在连接了少量磁盘的普通处理器上运行，而不是直接在磁盘控制处理器上运行，但基本方法类似。

我们的备份任务机制与 Charlotte 系统中使用的急切调度机制 [^3] 相似。简单急切调度的一个缺点是，如果某个任务不断失败，整个计算将无法完成。我们通过一种机制来跳过导致问题的记录，从而解决了这个问题的一些情况。

MapReduce 的实现依赖于一个内部的集群管理系统，该系统负责在大量共享机器上分发和运行用户任务。尽管这并非本文的核心内容，但这个集群管理系统在本质上与 Condor [^16] 等其他系统相似。

MapReduce 库中的排序功能在操作上与 NOW-Sort [^1] 类似。源机器（map 工作进程）对要排序的数据进行分区，并将其发送给 R 个 reduce 工作进程之一。每个 reduce 工作进程在其本地对数据进行排序（如果可能，在内存中）。当然，NOW-Sort 不具备用户可定义的 Map 和 Reduce 函数，这赋予了我们的库广泛的应用性。

River[^2]提供了一个编程模型，其中的进程通过在分布式队列上发送数据来相互通信。与 MapReduce 类似，River 系统旨在即使在异构硬件或系统扰动导致的非均匀性存在的情况下，也能提供良好的平均性能。River 通过精心调度磁盘和网络传输来实现任务均衡完成时间。MapReduce 采用了不同的方法。通过限制编程模型，MapReduce 框架能够将问题细分为大量小任务。这些任务在可用的工作节点上动态分配，使得更快的工作节点处理更多任务。受限的编程模型还允许我们在作业即将完成时安排任务的冗余执行，这在存在如慢速或卡住的工作节点等非均匀性的情况下，显著减少了完成时间。

BAD-FS [^5] 与 MapReduce 的编程模型有很大不同，且与 MapReduce 不同，BAD-FS 主要面向广域网中作业的执行。然而，两者有两个基本的相似点：(1) 两个系统都使用冗余执行来应对由于故障导致的数据丢失；(2) 两个系统都采用了感知局部性的调度方法，以减少通过拥塞网络链路传输的数据量。

TACC [^7] 是一个旨在简化构建高可用网络服务的系统。像 MapReduce 一样，TACC 依赖于重新执行作为实现容错的机制。

## 8 结论

MapReduce 编程模型已被 Google 成功应用于多种不同的用途。我们认为其成功有几个原因。首先，这个模型易于使用，即使是没有并行和分布式系统经验的程序员也能轻松上手，因为它隐藏了并行化、容错、局部性优化和负载均衡等细节。其次，许多问题可以很容易地表达为 MapReduce 计算。例如，MapReduce 被用于生成 Google 生产搜索服务的数据、排序、数据挖掘、机器学习以及许多其他系统。第三，我们开发了一个可扩展到大型机器集群的 MapReduce 实现，支持数千台机器。该实现能够高效利用这些机器资源，因此适合用于解决 Google 中遇到的许多大型计算问题。

从这项工作中，我们学到了几个重要的经验。首先，限制编程模型使得并行化和分布式计算变得更加容易，同时也能更轻松地实现容错。其次，网络带宽是一种稀缺资源。因此，我们系统中的许多优化都旨在减少通过网络传输的数据量：局部性优化使得我们能够从本地磁盘读取数据，而将中间数据写入本地磁盘的单一副本则节省了网络带宽。第三，冗余执行可以用来减少慢速机器的影响，并处理机器故障和数据丢失。

## 致谢

Josh Levenberg 凭借自己使用 MapReduce 的经验以及其他人的改进建议，在用户级 MapReduce API 中修改和扩展了许多新功能。MapReduce 从 Google 文件系统 [^8] 读取输入并将输出写入其中。我们要感谢 Mohit Aron、Howard Gobioff、Markus Gutschke、David Kramer、Shun-Tak Leung 和 Josh Redstone 在开发 GFS 方面所做的工作。我们还要感谢 Percy Liang 和 Olcan Sercinoglu 在开发 MapReduce 使用的集群管理系统方面所做的工作。Mike Burrows、Wilson Hsieh、Josh Levenberg、Sharon Perl、Rob Pike 和 Debby Wallach 对本文的早期草稿提出了有益的评论。匿名 OSDI 审阅者和我们的指导者 Eric Brewer 就本文的改进领域提出了许多有用的建议。最后，我们感谢 Google 工程组织内所有 MapReduce 用户提供有用的反馈、建议和错误报告。

## 附录A Word Frequency

本节包含一个程序，用于计算命令行指定的一组输入文件中每个唯一单词出现的次数。

```c++
#include "mapreduce/mapreduce.h"
// User’s map function
class WordCounter : public Mapper
{
public:
    virtual void Map(const MapInput &input)
    {
        const string &text = input.value();
        const int n = text.size();
        for (int i = 0; i < n;)
        {
            // Skip past leading whitespace
            while ((i < n) && isspace(text[i]))
                i++; // Find word end
            int start = i;
            while ((i < n) && !isspace(text[i]))
                i++;
            if (start < i)
                Emit(text.substr(start, i - start), "1");
        }
    }
};
REGISTER_MAPPER(WordCounter); // User’s reduce function
class Adder : public Reducer
{
    virtual void Reduce(ReduceInput *input)
    {
        // Iterate over all entries with the
        // same key and add the values
        int64 value = 0;
        while (!input->done())
        {
            value += StringToInt(input->value());
            input->NextValue();
        } 
        // Emit sum for input->key()
        Emit(IntToString(value));
    }
};
REGISTER_REDUCER(Adder);
int main(int argc, char **argv)
{
    ParseCommandLineFlags(argc, argv);
    MapReduceSpecification spec; // Store list of input files into "spec"
    for (int i = 1; i < argc; i++)
    {
        MapReduceInput *input = spec.add_input();
        input->set_format("text");
        input->set_filepattern(argv[i]);
        input->set_mapper_class("WordCounter");
    }
    // Specify the output files:
    // /gfs/test/freq-00000-of-00100
    // /gfs/test/freq-00001-of-00100
    // ...
    MapReduceOutput *out = spec.output();
    out->set_filebase("/gfs/test/freq");
    out->set_num_tasks(100);
    out->set_format("text");
    out->set_reducer_class("Adder");
    // Optional: do partial sums within map
    // tasks to save network bandwidth
    out->set_combiner_class("Adder");
    // Tuning parameters: use at most 2000
    // machines and 100 MB of memory per task
    spec.set_machines(2000);
    spec.set_map_megabytes(100);
    spec.set_reduce_megabytes(100);
    // Now run it MapReduceResult result;
    if (!MapReduce(spec, &result))
        abort();
    // Done: ’result’ structure contains info
    // about counters, time taken, number of
    // machines used, etc.
    return 0;
}
```

## 参考文献

[^1]: Andrea C. Arpaci-Dusseau, Remzi H. Arpaci-Dusseau, David E. Culler, Joseph M. Hellerstein, and David A. Patterson. High-performance sorting on networks of workstations. In Proceedings of the 1997 ACM SIGMOD International Conference on Management of Data, Tucson, Arizona, May 1997.

[^2]: Remzi H. Arpaci-Dusseau, Eric Anderson, Noah Treuhaft, David E. Culler, Joseph M. Hellerstein, David Patterson, and Kathy Yelick. Cluster I/O with River: Making the fast case common. In Proceedings of the Sixth Workshop on Input/Output in Parallel and Distributed Systems (IOPADS ’99), pages 10–22, Atlanta, Georgia, May 1999.

[^3]: Arash Baratloo, Mehmet Karaul, Zvi Kedem, and Peter Wyckoff. Charlotte: Metacomputing on the web. In Proceedings of the 9th International Conference on Parallel and Distributed Computing Systems, 1996.

[^4]: Luiz A. Barroso, Jeffrey Dean, and Urs Ho ̈lzle. Web search  for a planet: The Google cluster architecture. IEEE Micro, 23(2):22–28,  April 2003.

[^5]: John Bent, Douglas Thain, Andrea C.Arpaci-Dusseau, Remzi H. Arpaci-Dusseau, and Miron Livny. Explicit control in a batch-aware distributed file system. In Proceedings of the 1st USENIX Symposium on Networked Systems Design and Implementation NSDI, March 2004.

[^6]: Guy E. Blelloch. Scans as primitive parallel operations. IEEE Transactions on Computers, C-38(11), November 1989.

[^7]: Armando Fox, Steven D. Gribble, Yatin Chawathe, Eric A. Brewer, and Paul Gauthier. Cluster-based scalable network services. In Proceedings of the 16th ACM Symposium on Operating System Principles, pages 7891, Saint-Malo, France, 1997.

[^8]: Sanjay Ghemawat, Howard Gobioff, and Shun-Tak Leung. The  Google file system. In 19th Symposium on Operating Systems Principles,  pages 29–43, Lake George, New York, 2003.

[^9]: S. Gorlatch. Systematic efficient parallelization of scan and other list homomorphisms. In L. Bouge, P. Fraigniaud, A. Mignotte, and Y. Robert, editors, Euro-Par’96. Parallel Processing, Lecture Notes in Computer Science 1124, pages 401–408. Springer-Verlag, 1996.

[^10]: Jim Gray. Sort benchmark home page. [http://research.microsoft.com/barc/SortBenchmark/](http://research.microsoft.com/barc/SortBenchmark/).

[^11]: William Gropp, Ewing Lusk, and Anthony Skjellum. Using MPI: Portable Parallel Programming with the Message-Passing Interface. MIT Press, Cambridge, MA, 1999.

[^12]: L. Huston, R. Sukthankar, R. Wickremesinghe, M. Satyanarayanan, G. R. Ganger, E. Riedel, and A. Ailamaki. Diamond: A storage architecture for early discard in interactive search. In Proceedings of the 2004 USENIX File and Storage Technologies FAST Conference, April 2004.

[^13]: Richard E. Ladner and Michael J. Fischer. Parallel prefix computation. Journal of the ACM, 27(4):831–838, 1980.

[^14]: Michael O. Rabin. Efficient dispersal of information for security, load balancing and fault tolerance. Journal of the ACM, 36(2):335–348, 1989.  

[^15]: Erik Riedel, Christos Faloutsos, Garth A. Gibson, and David Nagle. Active disks for large-scale data processing. IEEE Computer, pages 68–74, June 2001.  

[^16]: Douglas Thain, Todd Tannenbaum, and Miron Livny. Distributed computing in practice: The Condor experience. Concurrency and Computation: Practice and Experience, 2004.  

[^17]: L. G. Valiant. A bridging model for parallel computation. Communications of the ACM, 33(8):103–111, 1997.  

[^18]: Jim Wyllie. Spsort: How to sort a terabyte quickly. [http://alme1.almaden.ibm.com/cs/spsort.pdf](http://alme1.almaden.ibm.com/cs/spsort.pdf).
