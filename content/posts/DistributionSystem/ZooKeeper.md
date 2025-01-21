---
title: "[论文翻译]ZooKeeper: Wait-free coordination for Internet-scale systems"
date: 2025-01-21
draft: false
tags: ["分布式", "论文"]
categories: ["Learning"]
---

----
<div style="display: flex; justify-content: space-between;">
  <span style="text-align: center; display: inline-block; width: 45%;">
    Patrick Hunt and Mahadev Konar<br>
    Yahoo! Grid<br>
    {phunt,mahadev}@yahoo-inc.com
  </span>
  <span style="text-align: center; display: inline-block; width: 45%;">
    Flavio P. Junqueira and Benjamin Reed<br>
    Yahoo! Research<br>
    {fpj,breed}@yahoo-inc.com
  </span>
</div>

----

## 摘要

在本文中，我们描述了Zookeeper，一种用于协调分布式应用程序进程的服务。由于Zookeeper是关键基础设施的一部分，它旨在为客户端构建更复杂的协调原语提供一个简单且高性能的核心。它在一个复制的集中式服务中结合了组消息传递、共享寄存器和分布式锁服务的元素。Zookeeper提供的接口具有共享寄存器的无等待特性，并带有一个类似于分布式文件系统中缓存失效的事件驱动机制，从而提供一个简单而强大的协调服务。

Zookeeper的接口支持高性能服务实现。除了无等待特性外，Zookeeper还为客户机提供了请求的先进先出（FIFO）执行保证，并对所有改变Zookeeper状态的请求提供线性一致性。这些设计决策使得实现高性能处理流水线成为可能，读请求可以由本地服务器满足。我们展示了针对目标工作负载（读写比例为2:1到100:1），Zookeeper每秒能够处理数万到数十万次事务。这种性能允许客户端应用程序广泛使用Zookeeper。

## 1 引言

大规模分布式应用程序需要不同形式的协调。配置是最基本的协调形式之一。在其最简单的形式中，配置只是系统进程的操作参数列表，而更复杂的系统则具有动态配置参数。组成员关系和领导者选举在分布式系统中也很常见：通常进程需要知道其他哪些进程是活跃的以及这些进程负责什么。锁是一种强大的协调原语，用于实现对关键资源的互斥访问。

一种协调的方法是为每种不同的协调需求开发特定的服务。例如，Amazon Simple Queue Service [^3] 专门针对队列设计。其他服务则专门为领导者选举 [^25] 和配置 [^27] 开发。实现更强大原语的服务可以用来实现功能较弱的原语。例如，Chubby [^6] 是一个具有强同步保证的锁定服务。然后可以使用锁来实现领导者选举、组成员关系等。

在设计我们的协调服务时，我们没有选择在服务器端实现特定的原语，而是选择公开一个API，使应用程序开发人员能够实现自己的原语。这种选择导致了**协调内核**（coordination kernel）的实现，使得无需更改服务核心即可启用新的原语。这种方法允许根据应用的需求适应多种协调形式，而不是限制开发者使用固定的一组原语。

在设计Zookeeper的API时，我们避开了锁等阻塞原语。协调服务中的阻塞原语可能会导致诸如慢速或故障客户端对更快客户端性能产生负面影响等问题。如果请求处理依赖于其他客户端的响应和故障检测，服务本身的实现也会变得更加复杂。因此，我们的系统Zookeeper实现了一个操作简单**无等待**（wait-free）数据对象的API，这些数据对象以类似于文件系统的方式进行层次化组织。实际上，Zookeeper的API与其他文件系统的API相似，仅从API签名来看，Zookeeper似乎就是没有锁方法、打开和关闭功能的Chubby。然而，实现无等待数据对象使Zookeeper与基于锁等阻塞原语的系统有显著区别。

尽管无等待特性对性能和容错至关重要，但对于协调来说，这还不够。我们还需要为操作提供顺序保证。特别是，我们发现保证所有操作的先进先出（FIFO）客户机顺序和写操作的线性一致性，能够实现服务的高效实施，并且足以实现对我们应用程序感兴趣的协调原语。实际上，我们可以使用我们的API为任意数量的进程实现共识，根据Herlihy的层次结构，Zookeeper实现了通用对象[^14]。

Zookeeper服务由一组使用复制技术以实现高可用性和性能的服务器组成。其高性能使得包含大量进程的应用程序能够使用这种协调内核来管理所有方面的协调。我们能够使用一种简单的流水线架构实现Zookeeper，这种架构允许我们在仍有低延迟的情况下处理数百或数千个未完成的请求。这样的流水线自然支持单个客户端操作按FIFO顺序执行。保证FIFO客户机顺序使客户机可以异步提交操作。通过异步操作，客户机能够同时拥有多个未完成的操作。例如，当一个新的客户机成为领导者并且需要相应地操作和更新元数据时，这一特性是理想的。如果没有多未完成操作的可能性，初始化时间可能会达到数秒而不是亚秒级。

为了保证更新操作满足线性一致性，我们实现了一个基于领导者的原子广播协议[^23]，称为Zab[^24]。然而，一个典型的Zookeeper应用程序工作负载主要由读操作主导，因此需要扩展读吞吐量。在Zookeeper中，服务器本地处理读操作，并且不使用Zab对它们进行全局排序。

客户端侧缓存数据是提高读性能的重要技术。例如，对于一个进程来说，缓存当前领导者的标识符比每次需要知道领导者信息时都查询Zookeeper要高效得多。Zookeeper使用观察机制使客户端能够在不直接管理客户端缓存的情况下缓存数据。通过这种机制，客户端可以监视给定数据对象的更新，并在发生更新时收到通知。Chubby直接管理客户端缓存，它阻止更新以使所有缓存了正在更改数据的客户端的缓存失效。在这种设计下，如果这些客户端中有任何一个缓慢或出现故障，更新将会被延迟。Chubby使用租约防止故障客户端无限期地阻塞系统。然而，租约仅限制了缓慢或故障客户端的影响，而Zookeeper的观察机制则完全避免了这一问题。

在本文中，我们讨论 ZooKeeper 的设计和实现。借助 ZooKeeper，我们能够实现应用程序所需的所有协调原语，即使只有写入是可线性化的。为了验证我们的方法，我们展示了如何使用 ZooKeeper 实现一些协调原语。

总之，本文的主要贡献如下：

1. **协调内核**：我们提出了一种具有宽松一致性保证的无等待协调服务，用于分布式系统中。特别是，我们描述了协调内核的设计与实现，该内核已在许多关键应用中用于实现各种协调技术。

2. **协调方案**：我们展示了如何使用Zookeeper构建更高级别的协调原语，甚至是阻塞和强一致性的原语，这些原语在分布式应用中经常被使用。

3. **协调经验**：我们分享了一些使用Zookeeper的方式，并评估了其性能。通过这些实际应用和性能评估，我们可以更好地理解Zookeeper在真实环境中的表现及其对分布式应用的支持能力。

## 2 ZooKeeper 服务

客户通过使用Zookeeper客户端库的客户端API向Zookeeper提交请求。除了通过客户端API暴露Zookeeper服务接口外，客户端库还管理客户端与Zookeeper服务器之间的网络连接。在本节中，我们首先提供Zookeeper服务的高层视图。然后讨论客户端用于与Zookeeper交互的API。

**术语。**在本文中，我们使用**客户端**（client）表示ZooKeeper服务的用户，**服务器**（server）表示提供ZooKeeper服务的进程，**znode**表示ZooKeeper数据中的内存数据节点，这些节点在一个分层命名空间中组织，称为**数据树**（data tree）。我们还使用术语更新和写入来指代任何修改数据树状态的操作。客户端在连接到ZooKeeper并获取会话句柄通过其发出请求时建立**会话**（session）。

### 2.1 服务总览

ZooKeeper为其客户端提供了一组按照分层命名空间组织的数据节点（znodes）的抽象。这个层次结构中的znodes是客户端通过ZooKeeper API进行操作的数据对象。分层命名空间常见于文件系统中，这是一种理想的组织数据对象的方式，因为用户对此抽象非常熟悉，并且它能够更好地组织应用程序元数据。为了引用一个特定的znode，我们使用标准的UNIX文件系统路径表示法。例如，我们使用/A/B/C来表示znode C的路径，其中C的父节点是B，而B的父节点是A。所有znodes都存储数据，并且除了临时znodes外，所有znodes都可以有子节点。

<img src="https://picgo-01.oss-cn-shanghai.aliyuncs.com/Images/zk01.png" alt="zk01" title="图1：ZooKeeper 分层名称空间的图示。" style="zoom:50%;" />

客户端可以创建两种类型的 znode：

1. **常规znodes**（regular）：客户端通过显式地创建和删除来操作常规znodes。

2. **临时znodes**（ephemeral）：客户端创建此类znodes，它们要么由客户端显式删除，要么在创建它们的会话终止（无论是故意终止还是由于故障）时由系统自动移除。

此外，当创建一个新的znode时，客户端可以设置一个**顺序**（sequential）标志。设置了顺序标志的节点会在其名称后附加一个单调递增计数器的值。如果$n$是新的znode，$p$是父znode，则$n$的序列值永远不会小于在$p$下创建的任何其他顺序znode名称中的值。

ZooKeeper实现了监视（watches）机制，允许客户端在不进行轮询的情况下及时接收变化的通知。当客户端发出带有监视标志的读操作时，操作会正常完成，但服务器承诺在返回的信息发生变化时通知客户端。监视是一次性触发器，与一个会话相关联；一旦触发或会话关闭，它们就会被注销。监视仅指示发生了变化，但不会提供具体的变化内容。例如，如果客户端在“/foo”被更改两次之前发出`getData(''/foo'', true)`，客户端将收到一个监视事件，告知“/foo”的数据已更改。会话事件，如连接丢失事件，也会发送到监视回调中，以便客户端知道监视事件可能会延迟。

**数据模型。** ZooKeeper的数据模型本质上是一个简化了API的文件系统，仅支持全数据读写，或者是一个具有层次键的键/值表。层次命名空间有助于为不同应用的命名空间分配子树，并设置这些子树的访问权限。我们还利用客户端侧的目录概念来构建更高级的原语，如我们在第2.4节中将看到的。与文件系统中的文件不同，znodes并不是为通用数据存储设计的。相反，znodes映射到客户端应用程序的抽象，通常对应于用于协调目的的元数据。

举例来说，在图1中，我们有两个子树，一个用于应用程序1（/app1），另一个用于应用程序2（/app2）。应用程序1的子树实现了一个简单的组成员协议：每个客户端进程$p_i$在/app1下创建一个znode `p_i`，只要该进程正在运行，这个znode就会持续存在。

尽管znodes并未设计用于通用数据存储，ZooKeeper确实允许客户端存储一些可用于分布式计算中的元数据或配置的信息。例如，在基于领导者的应用中，刚开始启动的应用服务器可以利用这种信息了解当前的领导者是哪一台服务器。为了实现这一目标，当前的领导者可以在znode空间中的已知位置写入这些信息。znodes还关联有时间戳和版本计数器等元数据，这使得客户端能够跟踪znodes的变化，并基于znode的版本执行条件更新。

**会话。** 客户端连接到ZooKeeper并启动一个会话。会话有一个关联的超时时间。如果ZooKeeper在超过该超时时间内未从会话中收到任何信息，则认为该客户端出现故障。当客户端显式关闭会话句柄或ZooKeeper检测到客户端出现故障时，会话结束。在一个会话中，客户端会观察到一系列状态变化，这些变化反映了其操作的执行情况。会话使得客户端能够在ZooKeeper集群中的服务器之间透明地切换，从而实现在ZooKeeper服务器之间的持久化。

### 2.2 Client API

我们下面展示ZooKeeper API的一个相关子集，并讨论每个请求的语义。

- `create(path, data, flags)`: 创建一个路径名为`path`的znode，将`data[]`存储在其中，并返回新znode的名称。`flags`使客户端可以选择znode的类型：常规、临时，并设置顺序标志。

- `delete(path, version)`: 如果znode处于预期版本，则删除路径为`path`的znode。

- `exists(path, watch)`: 如果路径名为`path`的znode存在，则返回true，否则返回false。`watch`标志使客户端能够在znode上设置监视。

- `getData(path, watch)`: 返回与znode关联的数据和元数据（如版本信息）。`watch`标志的作用方式与`exists()`相同，除非znode不存在时ZooKeeper不会设置监视。

- `setData(path, data, version)`: 如果版本号是znode的当前版本，则将`data[]`写入路径为`path`的znode。

- `getChildren(path, watch)`: 返回一个znode的所有子节点的名称集合。

- `sync(path)`: 等待在操作开始时所有挂起的更新传播到客户端所连接的服务器。路径目前被忽略。

所有方法在API中都有同步和异步两种版本。当应用程序需要执行单个ZooKeeper操作且没有其他并发任务要执行时，可以使用同步API，它会进行必要的ZooKeeper调用并阻塞。然而，异步API允许应用程序同时有多个未完成的ZooKeeper操作和其他任务并行执行。ZooKeeper客户端保证每个操作对应的回调按顺序被调用。

请注意，ZooKeeper不使用句柄来访问znodes。每个请求都包含正在操作的znode的完整路径。这一选择不仅简化了API（无需`open()`或`close()`方法），还消除了服务器需要维护的额外状态。

每个更新方法都需要一个预期的版本号，这使得实现条件更新成为可能。如果znode的实际版本号与预期版本号不匹配，则更新会因版本不匹配错误而失败。如果版本号为-1，则不执行版本检查。

### 2.3 ZooKeeper保证

ZooKeeper有两个基本的顺序保证：

1. **线性化写入**：所有更新ZooKeeper状态的请求都是可序列化的，并且遵循优先顺序。
2. **FIFO客户端顺序**：来自同一客户端的所有请求都按照客户端发送的顺序执行。

注意，我们对线性一致性的定义与Herlihy[^15]最初提出的定义不同，我们称之为**A-线性一致性**（A-linearizability）（异步线性一致性）。在Herlihy最初的线性一致性定义中，客户端一次只能有一个未完成的操作（即一个客户端相当于一个线程）。而在我们的定义中，我们允许客户端有多个未完成的操作，因此我们可以选择不对同一客户端的未完成操作保证任何特定顺序，或者保证FIFO顺序。我们选择了后者作为我们的属性。重要的是要观察到，所有适用于线性一致对象的结果也适用于A-线性一致对象，因为满足A-线性一致性的系统也满足线性一致性。由于只有更新请求是A-线性一致的，ZooKeeper在每个副本上本地处理读请求。这使得服务能够在添加服务器时线性扩展。

为了理解这两个保证如何相互作用，考虑以下场景：一个由多个进程组成的系统选举一个领导者来指挥工作进程。当一个新的领导者接管系统时，它必须更改大量的配置参数，并在完成后通知其他进程。这里我们有两个重要的要求：

- 当新领导者开始进行更改时，我们不希望其他进程开始使用正在更改的配置；
- 如果新领导者在配置完全更新之前死亡，我们不希望进程使用这个部分配置。

观察到分布式锁，如Chubby提供的锁，可以帮助满足第一个要求，但对于第二个要求是不够的。使用ZooKeeper，新的领导者可以指定一个路径作为**就绪**（ready） znode；其他进程只有在该znode存在时才会使用配置。新领导者通过删除就绪znode、更新各种配置znode并重新创建就绪来更改配置。所有这些更改都可以流水线化并异步发出以快速更新配置状态。尽管更改操作的延迟大约为2毫秒，但如果一个接一个地发出请求，新领导者必须更新5000个不同的znode将需要10秒钟；通过异步发出请求，整个过程将花费不到一秒钟。由于顺序保证，如果一个进程看到就绪 znode，它也必须看到新领导者做出的所有配置更改。如果新领导者在就绪 znode创建之前失败，其他进程就知道配置尚未最终确定并不会使用它。

上述方案仍然有一个问题：如果一个进程在新领导者开始进行更改之前看到就绪znode存在，然后在更改进行过程中开始读取配置，会发生什么？这个问题通过通知的顺序保证来解决：如果客户端正在监视更改，那么在更改完成后，客户端会在看到系统的新状态之前看到通知事件。因此，如果读取就绪znode的进程请求对该znode的更改通知，它将在读取任何新的配置之前看到通知，告知客户端发生了更改。

另一个问题可能出现在客户端除了使用ZooKeeper之外还有自己的通信渠道时。例如，考虑两个客户端A和B在ZooKeeper中有一个共享配置并通过一个共享通信渠道进行通信。如果A在ZooKeeper中更改了共享配置并通过共享通信渠道通知B，B会在重新读取配置时期望看到该更改。如果B的ZooKeeper副本稍微落后于A的副本，它可能不会立即看到新的配置。利用上述保证，B可以通过在重新读取配置之前发出一个写请求来确保看到最新的信息。为了更高效地处理这种情况，ZooKeeper提供了`sync`请求：当跟随一个读操作时，构成一个**慢读**（slow read）。`sync`会使得服务器在处理读请求之前应用所有挂起的写请求，而无需承担完整写入的开销。这个原语在概念上类似于ISIS [^5]中的`flush`原语。

ZooKeeper还具有以下两个活跃性和持久性保证：如果大多数ZooKeeper服务器处于活动状态并且能够相互通信，则服务将可用；如果ZooKeeper服务成功响应了一个更改请求，那么该更改将在任何数量的故障后依然存在，只要最终有足够数量的服务器能够恢复。

### 2.4 原语示例

在本节中，我们将展示如何使用ZooKeeper API实现更强大的原语。ZooKeeper服务对这些更强大的原语一无所知，因为它们完全是在客户端使用ZooKeeper客户端API实现的。一些常见的原语，如组成员关系和配置管理，也是无等待的。对于其他原语，如会合（rendezvous），客户端需要等待一个事件。尽管ZooKeeper是无等待的，我们仍然可以使用ZooKeeper实现高效的阻塞原语。ZooKeeper的顺序保证允许对系统状态进行高效推理，而监视机制则允许高效地等待。

**配置管理**。ZooKeeper可以用于在分布式应用中实现动态配置。最简单的情况下，配置存储在一个znode，$z_c$中。进程启动时使用$z_c$的完整路径名。启动的进程通过读取$z_c$并设置监视标志为true来获取其配置。如果$z_c$中的配置被更新，进程会收到通知，并再次读取新的配置，同时将监视标志设置为true。

注意，在此方案中，如同在大多数使用监视的其他方案中一样，监视用于确保进程拥有最新的信息。例如，如果一个正在监视$z_c$的进程收到$z_c$更改的通知，并且在它能够发出对$z_c$的读取请求之前，$z_c$又发生了三次更改，该进程不会收到额外的三个通知事件。这不会影响进程的行为，因为那三个事件只会通知进程它已经知道的事情：它所拥有的$z_c$信息已经过时了。

**会合**。在分布式系统中，有时事先并不清楚最终的系统配置会是什么样子。例如，客户端可能想要启动一个主进程和若干个工作进程，但启动这些进程是由调度器完成的，因此客户端事先不知道诸如地址和端口之类的信息，这些信息是工作进程连接到主进程所需要的。我们可以通过ZooKeeper使用一个会合znode，$z_r$来处理这种情况，$z_r$是由客户端创建的一个节点。客户端将$z_r$的完整路径名作为启动参数传递给主进程和工作进程。当主进程启动时，它会在$z_r$中填入其使用的地址和端口信息。当工作进程启动时，它们会读取$z_r$并将监视标志设置为true。如果$z_r$尚未填充，工作进程会等待在$z_r$更新时收到通知。如果$z_r$是一个临时节点，主进程和工作进程可以监视$z_r$是否被删除，并在客户端结束时自行清理。

**组成员关系**。我们利用临时节点来实现组成员关系。具体来说，我们利用了这样一个事实，即临时节点允许我们查看创建该节点的会话的状态。我们首先指定一个znode，$z_g$，来代表组。当组中的一个进程启动时，它会在$z_g$下创建一个临时子znode。如果每个进程都有唯一的名称或标识符，则使用该名称作为子znode的名称；否则，进程可以使用SEQUENTIAL标志创建znode以获得唯一的名称分配。进程可以在子znode的数据中放入进程信息，例如使用的地址和端口。

在$z_g$下创建子znode后，进程正常启动，不需要做其他事情。如果进程失败或结束，其在$z_g$下的znode将被自动移除。

进程可以通过简单地列出$z_g$的子节点来获取组信息。如果进程希望监视组成员关系的变化，它可以将监视标志设置为true，并在接收到更改通知时刷新组信息（始终将监视标志设置为true）。

**简单锁**。尽管ZooKeeper不是一个锁服务，但它可以用来实现锁。使用ZooKeeper的应用程序通常会使用根据其需求定制的同步原语，如上述所示。这里我们将展示如何使用ZooKeeper实现锁，以表明它可以实现各种通用的同步原语。

最简单的锁实现使用“锁文件”。锁由一个znode表示。要获取锁，客户端尝试使用EPHEMERAL标志创建指定的znode。如果创建成功，客户端就持有了锁。否则，客户端可以读取该znode并将监视标志设置为true，以便在当前持有者失效时收到通知。当客户端失效或显式删除znode时，它会释放锁。其他等待锁的客户端一旦观察到znode被删除，就会再次尝试获取锁。

虽然这种简单的锁定协议有效，但它确实存在一些问题。首先，它受到herd效应的影响。如果有许多客户端在等待获取锁，当锁释放时，它们都会竞争这个锁，即使最终只有一个客户端能够获得锁。其次，它仅实现了独占锁。接下来的两个原语展示了如何克服这些问题。

**简单锁（无 herd 效应）**。我们定义一个锁znode $l$来实现这种锁。直观地说，我们将所有请求锁的客户端排成一行，每个客户端按照请求到达的顺序依次获取锁。因此，希望获取锁的客户端执行以下操作：

`Lock`

```python
n = create(l + “/lock-”, EPHEMERAL|SEQUENTIAL)
C = getChildren(l, false)
if n is lowest znode in C, exit
p = znode in C ordered just before n
if exists(p, true) wait for watch event
goto 2
```

`Unlock`

```python
delete(n)
```

在`Lock`的第1行中使用SEQUENTIAL标志，按照所有其他尝试的顺序对客户端尝试获取锁进行排序。如果客户端的znode在第3行具有最小的序列号，则该客户端持有锁。否则，客户端等待其znode之前持有锁或将会在该客户端的znode之前获得锁的znode被删除。通过仅监视位于自己znode之前的那个znode，我们避免了herd效应，因为在锁释放或锁请求被放弃时只会唤醒一个进程。一旦客户端正在监视的znode消失，客户端必须检查它现在是否持有锁。（之前的锁请求可能已被放弃，且存在一个序列号更小的znode仍在等待或持有锁。）

释放锁非常简单，只需删除代表锁请求的znode $n$。通过在创建时使用EPHEMERAL标志，崩溃的进程会自动清理任何锁请求或释放它们可能持有的任何锁。

总结来说，这种锁定方案具有以下优点：

1. 删除一个znode只会唤醒一个客户端，因为每个znode仅被另一个客户端监视，因此我们不会遇到herd效应；
2. 不需要轮询或超时；
3. 由于我们实现锁定的方式，通过浏览ZooKeeper数据可以查看锁竞争的程度、中断锁并调试锁定问题。

**读/写锁**。为了实现读/写锁，我们稍微改变了锁定过程，并具有单独的读锁和写锁过程。解锁过程与全局锁定情况相同。

`Write Lock`

```python
n = create(l + “/write-”, EPHEMERAL|SEQUENTIAL)
C = getChildren(l, false)
if n is lowest znode in C, exit
p = znode in C ordered just before n
if exists(p, true) wait for event
goto 2
```

`Read Lock`

```python
n = create(l + “/read-”, EPHEMERAL|SEQUENTIAL)
C = getChildren(l, false)
if no write znodes lower than n in C, exit
p = write znode in C ordered just before n
if exists(p, true) wait for event
goto 3
```

这种锁过程与之前的锁略有不同。写锁仅在命名上有所不同。由于读锁可以共享，第3行和第4行稍有不同，因为只有较早的写锁znode会阻止客户端获取读锁。看起来当有多个客户端等待读锁并在序列号较低的“写”znode被删除时收到通知时，我们可能会遇到“herd效应”；实际上，这是一种期望的行为，所有这些等待读锁的客户端都应该被释放，因为它们现在可能已经获得了锁。

**双屏障**。双屏障使客户端能够同步计算的开始和结束。当足够多的进程（由屏障阈值定义）加入屏障时，进程开始它们的计算并在完成后离开屏障。我们用znode来表示ZooKeeper中的屏障，称为$b$。每个进程$p$在进入时向$b$注册（通过创建一个znode作为$b$的子节点），并在准备离开时取消注册（删除子节点）。当$b$的子znode数量超过屏障阈值时，进程可以进入屏障。当所有进程都移除了它们的子节点时，进程可以离开屏障。我们使用监视来高效地等待进入和退出条件被满足。要进入屏障，进程会监视$ b $是否存在`ready`子节点，该子节点将由导致子级数量超过屏障阈值的进程创建。要离开屏障，进程会监视特定子节点的消失，并且仅在该znode被删除后才检查退出条件。

## 3 ZooKeeper 应用程序

我们现在描述一些使用ZooKeeper的应用，并简要说明它们是如何使用它的。我们将每个示例中的原语以**粗体**显示。

**获取服务（Fetching Service）**。抓取是搜索引擎的重要组成部分，Yahoo!抓取了数十亿的网页文档。获取服务（Fetching Service, FS）是Yahoo!爬虫的一部分，目前已经在生产环境中使用。本质上，它有主进程指挥页面抓取进程。主进程向抓取器提供配置，抓取器则写回报告它们的状态和健康状况。使用ZooKeeper为FS带来的主要优势包括从主进程故障中恢复、保证在故障情况下依然可用以及将客户端与服务器解耦，使它们只需从ZooKeeper读取状态即可将其请求导向健康的服务器。因此，FS主要使用ZooKeeper来管理**配置元数据**（configuration metadata），尽管它也使用ZooKeeper来进行主进程选举（**领导者选举**）（leader election）。

<img src="https://picgo-01.oss-cn-shanghai.aliyuncs.com/Images/zk02.png" alt="zk02" title="图2：具有获取服务的一台 ZK 服务器的工作负载。每个点代表一秒的样本。" style="zoom: 50%;" />

图2显示了FS在三天期间内使用的ZooKeeper服务器的读取和写入流量。为了生成此图，我们统计了这段时间内每秒的操作数量，每个点对应那一秒内的操作数量。我们观察到，读取流量相比写入流量要高得多。在每秒操作速率高于1,000次的期间，读取与写入的比例在10:1到100:1之间变化。此工作负载中的读取操作按普遍程度递增顺序为：`getData()`、`getChildren()` 和 `exists()`。

**Katta**。Katta[^17] 是一个分布式索引器，使用ZooKeeper进行协调，它是非Yahoo!应用的一个例子。Katta通过分片来划分索引工作。主服务器将分片分配给从服务器并跟踪进度。从服务器可能会失败，因此主服务器必须在从服务器加入或离开时重新分配负载。主服务器也可能会失败，因此其他服务器必须准备好在主服务器失败时接管。Katta使用ZooKeeper来跟踪从服务器和主服务器的状态（**组成员关系**）（group membership），并处理主服务器故障转移（**领导者选举**）（leader elction）。Katta还使用ZooKeeper来跟踪和传播分片到从服务器的分配（**配置管理**）（configuration management）。

**Yahoo! Message Broker**。Yahoo! Message Broker（YMB）是一个分布式发布-订阅系统。该系统管理着成千上万个客户端可以向其发布消息和从中接收消息的主题。这些主题分布在一组服务器上以提供可扩展性。每个主题使用主备方案进行复制，确保消息被复制到两台机器上以保证可靠的消息传递。构成YMB的服务器使用无共享分布式架构，这使得协调对于正确操作至关重要。YMB使用ZooKeeper来管理主题的分布（**配置元数据**），处理系统中机器的故障（**故障检测**和**组成员关系**）（failure detection），以及控制系统操作。

<img src="https://picgo-01.oss-cn-shanghai.aliyuncs.com/Images/zk03.png" alt="zk03" title="图3：YMB在ZooKeeper 中的结构布局" style="zoom:60%;" />

图3显示了YMB的部分znode数据布局。每个代理域都有一个名为`nodes`的znode，其中包含构成YMB服务的每个活动服务器的临时znode。每个YMB服务器在`nodes`下创建一个包含负载和状态信息的临时znode，通过ZooKeeper提供组成员关系和状态信息。诸如`shutdown`和`migration_prohibited`之类的节点被构成服务的所有服务器监控，允许对YMB进行集中控制。`topics`目录为YMB管理的每个主题包含一个子znode。这些主题znode有子znode，指示每个主题的主服务器和备份服务器以及该主题的订阅者。`primary`服务器和`backup`服务器的znode不仅允许服务器发现负责某个主题的服务器，还管理**领导者选举**和服务器崩溃的情况。

## 4 ZooKeeper实现

ZooKeeper通过在构成服务的每台服务器上复制ZooKeeper数据来提供高可用性。我们假设服务器通过崩溃失败，且这些故障服务器可能在之后恢复。图4展示了ZooKeeper服务的高层组件。当接收到请求时，服务器准备执行该请求（请求处理器）。如果该请求需要服务器之间的协调（写请求），则它们使用一个协议（原子广播的实现），最后服务器将更改提交到完全复制在所有服务器上的ZooKeeper数据库。对于读请求，服务器只需读取本地数据库的状态并生成对该请求的响应。

<img src="https://picgo-01.oss-cn-shanghai.aliyuncs.com/Images/zk04.png" alt="zk04" title="图4：ZooKeeper服务的组件" style="zoom:60%;" />

复制的数据库是一个**内存**（in-memory）数据库，包含整个数据树。树中的每个znode默认存储最多1MB的数据，但这个最大值是一个配置参数，在特定情况下可以更改。为了可恢复性，我们将更新高效地记录到磁盘，并在将写操作应用于内存数据库之前强制将其写入磁盘介质。实际上，像Chubby[^8]一样，我们保持一个已提交操作的重放日志（在我们的情况下是预写日志）并定期生成内存数据库的快照。

每个ZooKeeper服务器都为客户端提供服务。客户端连接到一个服务器来提交其请求。如前所述，读请求由每个服务器数据库的本地副本服务。更改服务状态的请求（写请求）则通过一个协议进行处理。

作为协议的一部分，写请求被转发到一个称为**领导者**（leader）的服务器。其余的ZooKeeper服务器，称为**跟随者**（followers），从领导者接收包含状态更改的消息提案，并就状态更改达成一致。

### 4.1 请求处理器

由于消息层是原子性的，我们保证本地副本永远不会出现分歧，尽管在任何时刻某些服务器可能已经应用了比其他服务器更多的事务。与客户端发送的请求不同，这些事务是**幂等的**（idempotent）。当领导者接收到一个写请求时，它会计算出当写操作被应用时系统的状态，并将其转换为一个捕获这一新状态的事务。未来状态必须被计算出来，因为可能存在尚未应用于数据库的未完成事务。例如，如果客户端执行了一个条件性的`setData`操作，并且请求中的版本号与即将更新的znode的未来版本号匹配，服务会生成一个包含新数据、新版本号和更新时间戳的`setDataTXN`。如果发生错误，例如版本号不匹配或要更新的znode不存在，则会生成一个`errorTXN`。

### 4.2 原子广播

所有更新ZooKeeper状态的请求都会被转发给领导者。领导者执行请求并通过Zab[^24]（一个原子广播协议）将状态更改广播到ZooKeeper状态。接收客户端请求的服务器在交付相应的状态更改时响应客户端。Zab默认使用简单多数派法定人数来决定提案，因此Zab和ZooKeeper只能在大多数服务器正常工作的情况下运行（即，有$2f + 1$台服务器时，可以容忍$f$次故障）。

为了实现高吞吐量，ZooKeeper尝试保持请求处理管道满载，可能会有数千个请求处于处理管道的不同部分。由于状态更改依赖于之前状态更改的应用，Zab提供了比常规原子广播更强的顺序保证。更具体地说，Zab保证由领导者广播的更改按照发送顺序交付，并且在领导者广播自己的更改之前，所有来自前一个领导者的更改都会交付给该领导者。

有一些实现细节简化了我们的实现并提供了卓越的性能。我们使用TCP作为传输协议，因此消息顺序由网络维护，这使我们能够简化实现。我们使用Zab选择的领导者作为ZooKeeper的领导者，这样创建事务的同一进程也负责提出这些事务。我们使用日志来跟踪提案，将其作为内存数据库的预写日志，因此我们不需要将消息两次写入磁盘。

在正常操作期间，Zab确实按顺序且仅传递一次所有消息，但由于Zab不会持久记录每个已传递消息的id，在恢复过程中Zab可能会重新传递某些消息。因为我们使用幂等事务，只要消息按顺序传递，多次传递是可以接受的。实际上，ZooKeeper要求Zab至少重新传递上一个快照开始之后已传递的所有消息。

### 4.3 复制数据库

每个副本在内存中都有ZooKeeper状态的一份副本。当ZooKeeper服务器从崩溃中恢复时，它需要恢复此内部状态。重放所有已传递的消息以恢复状态会在服务器运行一段时间后花费过长时间，因此ZooKeeper使用周期性的快照，并仅要求重新传递自快照开始以来的消息。我们将ZooKeeper快照称为**模糊快照**（fuzzy snapshots），因为我们不会锁定ZooKeeper状态来获取快照；相反，我们对树进行深度优先扫描，原子地读取每个znode的数据和元数据并将它们写入磁盘。由于生成的模糊快照可能应用了在快照生成过程中已传递的状态变化的一部分，结果可能不对应于任何时间点上的ZooKeeper状态。然而，由于状态变化是幂等的，只要按顺序应用状态变化，我们可以多次应用它们。

例如，假设在ZooKeeper数据树中，两个节点`/foo`和`/goo`分别具有值`f1`和`g1`，并且当模糊快照开始时两者都处于版本1，并且以下状态更改流到达，其形式为`<transactionType, path , value, new-version>`: 

```
<SetDataTXN, /foo, f2, 2>
<SetDataTXN, /goo, g2, 2>
<SetDataTXN, /foo, f3, 3>
```

处理这些状态变化后，`/foo` 和 `/goo` 的值分别为 `f3` 和 `g2`，版本分别为 3 和 2。然而，模糊快照可能记录了`/foo` 和 `/goo` 的值为 `f3` 和 `g1`，版本分别为 3 和 1，这并不是ZooKeeper数据树的一个有效状态。如果服务器在此快照下崩溃并恢复，并且Zab重新传递这些状态变化，最终的状态将与崩溃前的服务状态相对应。

### 4.4 客户端-服务器交互

当服务器处理写请求时，它还会发送并清除与该更新相关的所有通知。服务器按顺序处理写操作，并且不会与其他写或读操作并发执行。这确保了通知的严格顺序。需要注意的是，服务器在本地处理通知。只有客户端所连接的服务器才会跟踪并触发该客户端的通知。

读请求在每个服务器上本地处理。每个读请求都会被处理并标记一个**zxid**，该zxid对应于服务器看到的最后一个事务。这个zxid定义了读请求相对于写请求的部分顺序。通过在本地处理读取，我们获得了极佳的读性能，因为这只是本地服务器上的内存操作，没有磁盘活动或需要运行的协议。这一设计选择对于实现我们在读取主导的工作负载下达到卓越性能的目标至关重要。

使用快速读取的一个缺点是不能保证读操作的优先顺序。也就是说，读操作可能会返回一个过时的值，即使对同一znode的最近更新已经提交。并不是所有的应用程序都需要优先顺序，但对于确实需要这一点的应用程序，我们实现了`sync`操作。这个原语异步执行，并在所有待处理的写入到其本地副本后由领导者排序。为了保证特定读操作返回最新更新的值，客户端会在读操作前调用`sync`。客户端操作的FIFO顺序保证加上`sync`的全局保证，使得读操作的结果能够反映在`sync`发出之前发生的任何变化。在我们的实现中，我们不需要原子广播`sync`，因为我们使用的是基于领导者的算法，只需将`sync`操作放在领导者和执行`sync`调用的服务器之间的请求队列末尾即可。为了使此机制有效，跟随者必须确保领导者仍然是领导者。如果有未完成的事务正在提交，则服务器不会怀疑领导者的身份。如果待处理队列为空，领导者需要发起一个空事务来提交并将`sync`操作排在此事务之后。这种做法的好处是，当领导者负载较重时，不会产生额外的广播流量。在我们的实现中，超时时间设置为让领导者在跟随者放弃他们之前意识到自己不再是领导者，因此无需发起空事务。

ZooKeeper服务器按照FIFO顺序处理来自客户端的请求。响应中包含响应所对应的zxid。即使在没有活动的时间间隔内的心跳消息也包括客户端所连接服务器最后看到的zxid。如果客户端连接到一个新的服务器，该新服务器通过检查客户端的最后一个zxid与其自身的最后一个zxid来确保其对ZooKeeper数据的视图至少与客户端一样新。如果客户端的视图比服务器更近，服务器会在追赶上之前不会重新建立与客户端的会话。由于客户端只能看到已被复制到大多数ZooKeeper服务器的更改，因此可以保证客户端能够找到另一个具有系统最近视图的服务器。这种行为对于保证持久性非常重要。

为了检测客户端会话故障，ZooKeeper使用超时机制。如果在会话超时时间内没有任何服务器收到来自客户端会话的任何消息，领导者将判定发生了故障。如果客户端足够频繁地发送请求，则无需发送其他消息。否则，在活动较少的期间，客户端会发送心跳消息。如果客户端无法与服务器通信以发送请求或心跳，它会连接到不同的ZooKeeper服务器以重新建立其会话。为了避免会话超时，ZooKeeper客户端库会在会话空闲$s/3$毫秒后发送心跳，并且如果在$2s/3$毫秒内未从服务器收到消息，则切换到新的服务器，其中$s$是会话超时时间（以毫秒为单位）。

## 5 评估

### 5.1 吞吐量

为了评估我们的系统，我们对系统饱和时的吞吐量以及各种注入故障对吞吐量的影响进行了基准测试。我们改变了构成ZooKeeper服务的服务器数量，但始终保持客户端数量不变。为了模拟大量客户端，我们使用35台机器模拟250个同时在线的客户端。

我们有一个ZooKeeper服务器的Java实现，以及Java和C两种客户端（该实现可在 http://hadoop.apache.org/zookeeper 上公开获取）。在这些实验中，我们使用的Java服务器配置为在一个专用磁盘上记录日志，并在另一个磁盘上进行快照。我们的基准客户端使用异步Java客户端API，每个客户端至少有100个未完成的请求。每个请求包含读取或写入1K的数据。我们没有展示其他操作的基准测试结果，因为所有修改状态的操作性能大致相同，不修改状态的操作（除`sync`外）性能也大致相同。（`sync`的性能接近于轻量级写操作，因为请求必须发送到领导者，但不需要广播。）客户端每300毫秒发送一次已完成操作的数量统计，我们每6秒采样一次。为了避免内存溢出，服务器会限制系统中的并发请求数。ZooKeeper使用请求节流来防止服务器过载。对于这些实验，我们将ZooKeeper服务器配置为最多处理2000个总请求。

<img src="https://picgo-01.oss-cn-shanghai.aliyuncs.com/Images/zk05.png" alt="zk05" title="图 5：随着读取与写入的比率变化，饱和系统的吞吐量性能。" style="zoom:67%;" />

<img src="https://picgo-01.oss-cn-shanghai.aliyuncs.com/Images/zkT01.png" alt="zkT01" title="表 1：饱和系统极端情况下的吞吐量性能。" style="zoom: 50%;" />

在图5中，我们展示了当我们改变读请求与写请求的比例时的吞吐量，每条曲线对应提供ZooKeeper服务的不同数量的服务器。表1显示了读负载极端情况下的数据。读取吞吐量高于写入吞吐量，因为读取不使用原子广播。图表还表明，服务器数量对广播协议的性能有负面影响。从这些图表中，我们可以观察到系统中服务器的数量不仅影响服务能够处理的故障数量，还影响服务能够处理的工作负载。请注意，三条服务器的曲线在大约60%处与其他曲线相交。这种情况并非三服务器配置独有，由于本地读取允许的并行性，所有配置都会出现这种情况。然而，在图中的其他配置中并未观察到这一点，因为我们为了可读性限制了最大y轴吞吐量。

写请求比读请求耗时更长有两个原因。首先，写请求必须通过原子广播，这需要额外的处理并增加请求的延迟。写请求处理时间较长的另一个原因是，服务器必须确保事务在向领导者发送确认之前已记录到非易失性存储中。原则上，这一要求可能显得过度，但为了在我们的生产系统中换取可靠性，我们牺牲了一部分性能，因为ZooKeeper构成了应用的基本事实。我们使用更多的服务器来容忍更多的故障。通过将ZooKeeper数据分区到多个ZooKeeper集群中来提高写入吞吐量。这种复制与分区之间的性能权衡之前已被Gray等人观察到[^12]。

<img src="https://picgo-01.oss-cn-shanghai.aliyuncs.com/Images/zk06.png" alt="zk06" title="图 6：饱和系统的吞吐量，当所有客户端连接到领导者时，读取与写入的比率发生变化。" style="zoom:67%;" />

ZooKeeper通过将负载分布在其构成服务的服务器上来实现如此高的吞吐量。由于我们放宽了一致性保证，因此可以进行负载分配。相反，Chubby客户端将所有请求直接发送给领导者。图6展示了如果不利用这种放松一致性保证的方式，并强制客户端仅连接到领导者时会发生什么。如预期的那样，对于读取为主的工作负载，吞吐量明显更低，但即使是写入为主的工作负载，吞吐量也较低。由服务客户端引起的额外CPU和网络负载影响了领导者协调提案广播的能力，进而对整体写入性能产生了负面影响。

<img src="https://picgo-01.oss-cn-shanghai.aliyuncs.com/Images/zk07.png" alt="zk07" title="图 7：孤立的原子广播组件的平均吞吐量。误差线表示最小值和最大值。" style="zoom:67%;" />

原子广播协议完成了系统中的大部分工作，因此比任何其他组件更限制ZooKeeper的性能。图7显示了原子广播组件的吞吐量。为了对其性能进行基准测试，我们在领导者处直接生成事务来模拟客户端，因此没有客户端连接或客户端请求和响应。在最大吞吐量下，原子广播组件成为CPU受限的。理论上，图7的性能应该与100%写入情况下的ZooKeeper性能相匹配。然而，ZooKeeper的客户端通信、ACL检查以及请求到事务的转换都需要CPU。对CPU的竞争降低了ZooKeeper的整体吞吐量，使其远低于单独的原子广播组件。由于ZooKeeper是一个关键的生产组件，迄今为止我们的开发重点是正确性和鲁棒性。通过消除诸如额外副本、同一对象的多次序列化、更高效的内部数据结构等，有大量机会显著提高性能。

<img src="https://picgo-01.oss-cn-shanghai.aliyuncs.com/Images/zk08.png" alt="zk08" title="图 8：失败时的吞吐量。" style="zoom:67%;" />

为了展示在注入故障时系统随时间的行为，我们运行了一个由5台机器组成的ZooKeeper服务。我们运行了与之前相同的饱和基准测试，但这次我们将写入比例保持在一个恒定的30%，这是我们预期工作负载的一个保守比例。周期性地，我们会终止一些服务器进程。图8展示了系统吞吐量随着时间的变化情况。图中标记的事件如下：

1. 一个跟随者的故障和恢复；
2. 另一个跟随者的故障和恢复；
3. 领导者的故障；
4. 在前两个标记中两个跟随者（a, b）的故障，在第三个标记时恢复（c）；
5. 领导者的故障；
6. 领导者的恢复。

从这个图表中可以得出几个重要的观察结果。首先，如果跟随者快速失败并恢复，那么尽管发生了故障，ZooKeeper仍然能够维持较高的吞吐量。单个跟随者的故障不会阻止服务器形成法定人数，并且仅大致减少了该服务器故障前处理的读请求所占的比例。其次，我们的领导者选举算法能够足够快地恢复，以防止吞吐量大幅下降。根据我们的观察，ZooKeeper选举新领导者所需的时间不到200毫秒。因此，虽然服务器在短暂时间内停止服务请求，但由于我们的采样周期是按秒计算的，我们没有观察到吞吐量降为零的情况。第三，即使跟随者需要更多时间来恢复，一旦它们开始处理请求，ZooKeeper也能够再次提升吞吐量。事件1、2和4之后未能完全恢复到最大吞吐量的原因之一是，客户端只有在其与跟随者的连接中断时才会切换跟随者。因此，在事件4之后，直到事件3和5中的领导者失败，客户端才重新分配自己。实际上，随着客户端的加入和离开，这种不平衡会随着时间自行解决。

### 5.2 请求延迟

为了评估请求的延迟，我们创建了一个基于Chubby基准测试的模型进行基准测试[^6]。我们创建了一个工作进程，它简单地发送一个创建请求，等待其完成，然后异步删除新节点，并开始下一个创建请求。我们相应地改变了工作进程的数量，在每次运行中，每个工作进程创建50000个节点。我们通过将完成的创建请求数除以所有工作进程完成总时间来计算吞吐量。

<img src="https://picgo-01.oss-cn-shanghai.aliyuncs.com/Images/zkT02.png" alt="zkT02" title="表 2：每秒处理的创建请求数。" style="zoom: 50%;" />

表2展示了我们的基准测试结果。创建请求包含1K的数据，而不是Chubby基准测试中的5字节，以便更好地符合我们的预期使用情况。即使有这些较大的请求，ZooKeeper的吞吐量仍然比已公布的Chubby吞吐量高出3倍以上。单个ZooKeeper工作进程基准测试的吞吐量表明，对于三台服务器，平均请求延迟为1.2毫秒；对于九台服务器，平均请求延迟为1.4毫秒。

### 5.3 屏障的性能

在这个实验中，我们依次执行多个屏障以评估使用ZooKeeper实现的原语的性能。对于给定数量的屏障$b$，每个客户端首先进入所有$b$个屏障，然后依次离开所有$b$个屏障。由于我们使用的是2.4节中的双屏障算法，客户端必须等待所有其他客户端执行完`enter()`过程后才能进行下一个调用（`leave()`类似）。

<img src="https://picgo-01.oss-cn-shanghai.aliyuncs.com/Images/zkT03.png" alt="zkT03" title="表 3：屏障实验的时间（以秒为单位）。每个点是每个客户完成五次以上运行的平均时间。" style="zoom:67%;" />

我们在表3中报告了实验结果。在这个实验中，我们有50、100和200个客户端依次进入数量为b的屏障，$b\in {200, 400, 800, 1600}$。尽管一个应用可以有数千个ZooKeeper客户端，但在每次协调操作中通常只有较小的一部分客户端参与，因为客户端经常根据应用的具体情况进行分组。

从这个实验中有两个有趣的观察结果：处理所有屏障所需的时间大致与屏障的数量成线性增长，这表明对数据树同一部分的并发访问没有产生任何意外延迟，并且延迟随着客户端数量的增加而按比例增加。这是因为我们没有使ZooKeeper服务达到饱和状态。实际上，我们观察到即使客户端以同步方式进行操作，屏障操作（进入和离开）的吞吐量在所有情况下仍保持在每秒1,950到3,100次操作之间。在ZooKeeper操作中，这相当于每秒10,700到17,000次操作的吞吐量值。在我们的实现中，读写比为4:1（80%的读操作），因此我们基准代码使用的吞吐量远低于ZooKeeper能够实现的原始吞吐量（根据图5，超过40,000次操作/秒）。这是因为客户端需要等待其他客户端完成操作。

## 6 相关工作

ZooKeeper旨在提供一种服务，以缓解分布式应用中进程协调的问题。为了实现这一目标，其设计借鉴了以往的协调服务、容错系统、分布式算法和文件系统的理念。

我们并不是第一个提出用于协调分布式应用程序系统的团队。一些早期系统提出了针对事务性应用的分布式锁服务[^13]，以及用于计算机集群间信息共享的服务[^19]。最近，Chubby提出了一种为分布式应用程序管理咨询锁的系统[^6]。Chubby与ZooKeeper共享多个目标，它也提供了类似文件系统的接口，并使用一致性协议保证副本的一致性。然而，ZooKeeper不是一个锁服务。它可以被客户端用来实现锁，但其API中没有锁操作。不同于Chubby，ZooKeeper允许客户端连接到任何ZooKeeper服务器，而不仅仅是领导者。由于ZooKeeper的一致性模型比Chubby更为宽松，ZooKeeper客户端可以使用其本地副本提供数据和管理监视。这使得ZooKeeper能够提供比Chubby更高的性能，从而使应用程序能够更广泛地利用ZooKeeper。

文献中提出了多种容错系统，旨在缓解构建容错分布式应用程序的问题。一个早期的系统是ISIS[^5]，它将抽象类型规范转化为容错分布式对象，从而使容错机制对用户透明。Horus[^30]和Ensemble[^31]是从ISIS发展而来的系统。ZooKeeper采纳了ISIS的虚拟同步概念。Totem在利用局域网硬件广播的架构中保证消息传递的全局顺序[^22]。ZooKeeper适用于各种网络拓扑结构，这促使我们依赖于服务器进程之间的TCP连接，而不假设任何特殊的拓扑结构或硬件特性。我们也不公开ZooKeeper内部使用的任何ensemble通信。

构建容错服务的一个重要技术是状态机复制[^26]，Paxos[^20]是一种允许异步系统中有效实现复制状态机的算法。我们使用了一种与Paxos共享某些特性的算法，但结合了共识所需的事务日志记录和数据树恢复所需的预写日志记录，以实现高效的实现。关于实用的拜占庭容错复制状态机协议已有若干提议[^7] [^10] [^18] [^1] [^28]。ZooKeeper不假设服务器可以是拜占庭式的，但我们确实采用了如校验和和合理性检查等机制来捕捉非恶意的拜占庭故障。Clement等人讨论了一种在不修改当前服务器代码库的情况下使ZooKeeper完全拜占庭容错的方法[^9]。到目前为止，我们在生产环境中尚未观察到使用完全拜占庭容错协议可能预防的故障[^29]。

Boxwood[^21]是一个使用分布式锁服务器的系统，它为应用程序提供更高层次的抽象，并依赖于基于Paxos的分布式锁服务。与Boxwood类似，ZooKeeper是用于构建分布式系统的组件，但ZooKeeper有更高的性能要求，并在客户端应用中得到了更广泛的应用。ZooKeeper暴露了较低级别的原语，应用程序可以使用这些原语来实现更高级别的原语。

尽管ZooKeeper类似于一个小文件系统，但它只提供了文件系统操作的一小部分，并增加了大多数文件系统不具备的功能，如顺序保证和条件写入。ZooKeeper的监视功能与AFS[^16]中的缓存回调在精神上相似。

Sinfonia[^2]引入了迷你事务，这是一种构建可扩展分布式系统的新范式。Sinfonia被设计用来存储应用数据，而ZooKeeper则存储应用元数据。ZooKeeper将其状态完全复制并保存在内存中，以实现高性能和一致的延迟。我们对类似文件系统操作和顺序的使用使得功能类似于迷你事务。Znode是一个方便的抽象，在其基础上我们添加了监视功能，这是Sinfonia所缺少的。Dynamo[^11]允许客户端在一个分布式的键值存储中获取和放置相对较小（小于1M）的数据量。与ZooKeeper不同的是，Dynamo的键空间不是分层的。Dynamo也不提供对写入的强大持久性和一致性保证，而是通过读取解决冲突。DepSpace[^4]使用一个元组空间来提供拜占庭容错服务。与ZooKeeper一样，DepSpace使用简单的服务器接口在客户端实现强大的同步原语。虽然DepSpace的性能远低于ZooKeeper，但它提供了更强的容错能力和保密性保证。

## 7 结论

ZooKeeper通过向客户端暴露无等待对象，采取了一种无等待的方式来解决分布式系统中进程协调的问题。我们发现ZooKeeper不仅对Yahoo!内部的多个应用有用，对外部应用也同样有价值。对于读取为主的工作负载，ZooKeeper通过使用带有监视的快速读取（均由本地副本提供服务）实现了每秒数十万次操作的吞吐量。尽管我们的读取和监视的一致性保证看起来较弱，但我们通过实际用例展示了这种组合允许我们在客户端实现高效且复杂的协调协议，即使读取不具有优先顺序，数据对象的实现也是无等待的。无等待属性对于高性能至关重要。

虽然我们只描述了少数几个应用案例，但实际中有许多其他应用也在使用ZooKeeper。我们认为其成功主要归因于其简单的接口和通过该接口可以实现的强大抽象。由于ZooKeeper的高吞吐量，应用程序不仅可以使用它来进行粗粒度的锁定，还可以广泛地利用它。

## 致谢

我们想要感谢Andrew Kornev和Runping Qi对ZooKeeper的贡献；感谢Zeke Huang和Mark Marchukov提供的宝贵反馈；感谢Brian Cooper和Laurence Ramontianu在ZooKeeper早期开发阶段的贡献；同时感谢Brian Bershad和Geoff Voelker对展示内容提出的重要意见。

## 参考文献

[^1]: M. Abd-El-Malek, G. R. Ganger, G. R. Goodson, M. K. Reiter, and J. J. Wylie. Fault-scalable byzantine fault-tolerant services. In *SOSP ’05: Proceedings of the twentieth ACM symposium on Operating systems principles*, pages 59–74, New York, NY, USA, 2005. ACM.  
[^2]: M. Aguilera, A. Merchant, M. Shah, A. Veitch, and C. Karamanolis. Sinfonia: A new paradigm for building scalable distributed systems. In *SOSP ’07: Proceedings of the 21st ACM symposium on Operating systems principles*, New York, NY, 2007.  
[^3]: Amazon. Amazon simple queue service. http://aws.amazon.com/sqs/, 2008.  
[^4]: A. N. Bessani, E. P. Alchieri, M. Correia, and J. da Silva Fraga. Depspace: A byzantine fault-tolerant coordination service. In *Proceedings of the 3rd ACM SIGOPS/EuroSys European Systems Conference - EuroSys 2008*, Apr. 2008.  
[^5]: K. P. Birman. Replication and fault-tolerance in the ISIS system. In *SOSP ’85: Proceedings of the 10th ACM symposium on Operating systems principles*, New York, USA, 1985. ACM Press.  
[^6]: M. Burrows. The Chubby lock service for loosely-coupled distributed systems. In *Proceedings of the 7th ACM/USENIX Symposium on Operating Systems Design and Implementation (OSDI)*, 2006.  
[^7]: M. Castro and B. Liskov. Practical byzantine fault tolerance and proactive recovery. *ACM Transactions on Computer Systems*, 20(4), 2002.  
[^8]: T. Chandra, R. Griesemer, and J. Redstone. Paxos made live: An engineering perspective. In *Proceedings of the 26th annual ACM symposium on Principles of distributed computing (PODC)*, Aug. 2007.  
[^9]: A. Clement, M. Kapritsos, S. Lee, Y. Wang, L. Alvisi, M. Dahlin, and T. Riche. UpRight cluster services. In Proceedings of the 22 nd ACM Symposium on Operating Systems Principles (SOSP), Oct. 2009.  
[^10]: J. Cowling, D. Myers, B. Liskov, R. Rodrigues, and L. Shira. Hq replication: A hybrid quorum protocol for byzantine fault tolerance. In *SOSP ’07: Proceedings of the 21st ACM symposium on Operating systems principles*, New York, NY, USA, 2007.  
[^11]: G. DeCandia, D. Hastorun, M. Jampani, G. Kakulapati, A. Lakshman, A. Pilchin, S. Sivasubramanian, P. Vosshall, and W. Vogels. Dynamo: Amazons highly available key-value store. In  *SOSP ’07: Proceedings of the 21st ACM symposium on Operating systems principles*, New York, NY, USA, 2007. ACM Press.  
[^12]: J. Gray, P. Helland, P. O’Neil, and D. Shasha. The dangers of replication and a solution. In *Proceedings of SIGMOD ’96*, pages 173–182, New York, NY, USA, 1996. ACM.  
[^13]: A. Hastings. Distributed lock management in a transaction processing environment. In *Proceedings of IEEE 9th Symposium on Reliable Distributed Systems*, Oct. 1990.  
[^14]: M. Herlihy. Wait-free synchronization. *ACM Transactions on Programming Languages and Systems*, 13(1), 1991.  
[^15]: M. Herlihy and J. Wing. Linearizability: A correctness condition for concurrent objects. *ACM Transactions on Programming Languages and Systems*, 12(3), July 1990.  
[^16]: J. H. Howard, M. L. Kazar, S. G. Menees, D. A. Nichols, M. Satyanarayanan, R. N. Sidebotham, and M. J. West. Scale and performance in a distributed file system. *ACM Trans. Comput. Syst.*, 6(1), 1988. 
[^17]: Katta. Katta - distribute lucene indexes in a grid. http://katta.wiki.sourceforge.net/, 2008.  
[^18]: R. Kotla, L. Alvisi, M. Dahlin, A. Clement, and E. Wong. Zyzzyva: speculative byzantine fault tolerance. *SIGOPS Oper. Syst. Rev.*, 41(6):45–58, 2007.  
[^19]: N. P. Kronenberg, H. M. Levy, and W. D. Strecker. Vaxclusters (extended abstract): a closely-coupled distributed system. *SIGOPS Oper. Syst. Rev.*, 19(5), 1985.  
[^20]: L. Lamport. The part-time parliament. *ACM Transactions on Computer Systems*, 16(2), May 1998.  
[^21]: J. MacCormick, N. Murphy, M. Najork, C. A. Thekkath, and L. Zhou. Boxwood: Abstractions as the foundation for storage infrastructure. In *Proceedings of the 6th ACM/USENIX Symposium on Operating Systems Design and Implementation (OSDI)*, 2004.  
[^22]: L. Moser, P. Melliar-Smith, D. Agarwal, R. Budhia, C. LingleyPapadopoulos, and T. Archambault. The totem system. In *Proceedings of the 25th International Symposium on Fault-Tolerant Computing*, June 1995. 
[^23]: S. Mullender, editor. *Distributed Systems, 2nd edition.* ACM Press, New York, NY, USA, 1993.  
[^24]: B. Reed and F. P. Junqueira. A simple totally ordered broadcast protocol. In *LADIS ’08: Proceedings of the 2nd Workshop on Large-Scale Distributed Systems and Middleware*, pages 1–6, New York, NY, USA, 2008. ACM.  
[^25]: N. Schiper and S. Toueg. A robust and lightweight stable leader election service for dynamic systems. In DSN, 2008.  
[^26]: F. B. Schneider. Implementing fault-tolerant services using the state machine approach: A tutorial. *ACM Computing Surveys*, 22(4), 1990.  
[^27]: A. Sherman, P. A. Lisiecki, A. Berkheimer, and J. Wein. ACMS: The Akamai configuration management system. In *NSDI*, 2005.  
[^28]: A. Singh, P. Fonseca, P. Kuznetsov, R. Rodrigues, and P. Maniatis. Zeno: eventually consistent byzantine-fault tolerance. In *NSDI’09: Proceedings of the 6th USENIX symposium on Networked systems design and implementation*, pages 169–184, Berkeley, CA, USA, 2009. USENIX Association. 
[^29]: Y. J. Song, F. Junqueira, and B. Reed. BFT for the skeptics. http://www.net.t-labs.tu-berlin.de/~petr/BFTW3/abstracts/talk-abstract.pdf.  
[^30]: R. van Renesse and K. Birman. Horus, a flexible group communication systems. *Communications of the ACM*, 39(16), Apr. 1996.  
[^31]: R. van Renesse, K. Birman, M. Hayden, A. Vaysburd, and D. Karr. Building adaptive systems using ensemble. *Software - Practice and Experience*, 28(5), July 1998.
