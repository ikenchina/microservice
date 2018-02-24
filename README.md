## 学习笔记
### dapper-Google分布式跟踪系统论文
#### 1. 背景
分布式系统在生产环境中运行可能存在一些耗时请求，我们可能很难定位，这里有三种情况:
1. 工程师无法准确地定位到这次全局搜索是调用了哪些服务，因为新的服务、或者服务上的某些片段，都有可能在任何时间上线或者修改过。
2. 不能苛求工程师对所有参与这次全局搜索的服务都了如指掌，每个服务都有可能是由不同的团队开发或维护的。
3. 这些暴露出来的服务或者服务器有可能同时被其他客户端使用着，所以这些全局搜索的性能问题可能是由其他应用造成的。

上面这些情况都是可能出现的，所以针对Dapper，我们只有两点要求：
1. 无处不在的部署
2. 持续的监控

上面两点如果做到了，这我们可以对分布式系统中的任何服务都能做到持续监控和报表输出，以及监控告警

针对上面的两点要求，我们可以直接推出三个具体的设计目标：
1. 低消耗：跟踪系统对在线服务的影响做到足够小。
2. 应用级的透明：对于应用的开发者来说，是不需要知道有跟踪系统这回事的。应用透明，才可以无所不在的透明侵入
3. 延展性：Google至少在未来几年的服务和集群的规模下，监控系统都应该能完全把控住

一个额外的设计目标是为跟踪数据产生之后，进行分析的速度要快，理想情况是数据存入跟踪仓库后一分钟内就能统计出来。

做到真正的应用级别的透明，这应该是当下面临的最有挑战性的设计目标。具体做法：*我们把核心跟踪代码做得很轻巧，然后把它植入到哪些无所不在的公共组件中，比如：线程调用、控制流以及RPC库。* 我们发现，Dapper数据往往侧重性能方面的调查。

Dapper吸收了一些优秀文章(Pinpoint, Magpie和X-Trace)的设计理念, 同时提出了做出了新的贡献。例如：要想实现低损耗，采用率是很必要的, 即便是1/1000的采用率，对于跟踪数据的使用层面上，也可以提供足够多的信息。

#### 2. 设计方法
分布式跟踪系统需要记录在一次特定请求后，系统完成的所有工作信息。例如：前端A、两个中间层BC，以及两个后端DE。当一个用户发起一个请求时，首先到达前端，然后发送两个RPC到中间层BC，B立即返回，但是C需要和DE交互后再返回A，由A来响应最初的请求。

*对于这样一个请求，简单使用的分布式跟踪实现，就是为各个服务上每一次你发送和接收动作，来收集跟踪标识符(message identifiers)和时间戳(timestamp)*, 前者用于调用链跟踪，后者表示各个节点的时间开销

为了将所有记录条目与一个给定的发起者关联上并记录所有信息，现在有两种解决方案：1. 黑盒(black-box)和基于标注(annotation-based)的监控方案。
1. 黑盒方案：假定需要跟踪的除了上述信息之外没有额外的信息，这样使用统计回归技术来推断两者之间的关系。
2. 基于标注的方案：依赖于应用程序或者中间件明确地标记一个全局ID，从而链接每一条记录和发起者的请求。

*两者比较，黑盒方案比标注方案更轻便，他们需要更多的数据，以获得足够的精度，因为它们依赖于统计推论。标注方案最主要的缺点是，需要代码植入。*

我们倾向于认为，Dapper跟踪架构像是内嵌在RPC调用的树形结构。然而，我们的核心数据模型不只局限于我们特性RPC框架，我们还能跟踪其他行为。从形式上看，我们的Dapper跟踪模型使用的树形结构，Span以及Annotation

##### 2.1 跟踪树和span
在Dapper跟踪树结构中，树节点是整个架构的基本单元，而每个节点又是对span的引用。节点之间的连线表示该节点span和它的父span直接的关系。虽然*span在日志文件中只是简单的代表span的开始和结束时间，他们在整个树形结构中却是相对独立的。* 

```
|------------------------------time----------------------------------------------->
|     |-----------------------------------------------------------------|         |
|-----|      Frontend.Request[no parent id, and span_id: 1]             |---------|
|     |_________________________________________________________________|         |
|         |         backend.Call           |                                      |
|---------| [parent id: 1, and span_id: 2] |--------------------------------------|
|         |________________________________| _________________________________    |
|                                           |      backend.DoSomething        |   |
|-------------------------------------------|    [parent id: 1, span_id: 3 ]  |---|
|                                           |_________________________________|   |
|                                             |        helper.Call            |   |
|---------------------------------------------|    [parent id: 3, span_id: 4] |---|
|                                             |_______________________________|   |
|                                              |          helper.Call         |   |
|----------------------------------------------|   [parent id: 3, span_id: 5] |---|
|                                              |______________________________|   |
|---20-----22------24------26------28-------30-------32------34-----36--------38--|
```

上面这幅图说明了span在一个大的跟踪过程中是什么样的。Dapper记录了span名称，以及每个span的ID和父ID，已重建在一次追踪过程中不同span之间的关系。如果一个span没有父ID被称为root span。所有span都挂在 一个特定的跟踪上，也共用一个跟踪ID。所以这些ID用全局唯一的64位证书标示。在一个典型的Dapper跟踪中，我们希望为每一个RPC对应到一个单一的span上，而且每个额外的组件层都对应一个跟踪树形结构的层级。

![上幅图所示的一个单独span的细节图](span.jpg)

上图给出了一个更详细的典型Dapper跟踪span记录点视图。它表示helper.Call的RPC(分别为server端和client端). span的开始时间和结束时间，以及任何RPC的时间信息都通过Dapper在RPC组件库的植入记录下来。如果应用程序开发者选择在跟踪中增加他们自己的注释(如图中"foo"的注释)（业务数据），这些信息也会和其他span信息一样记录下来。

记住，任何一个span可以包含来自不同的主机信息，这些也要记录下来。事实上，每一个RPC span可以包含客户端和服务器两个过程的注释，使得连接两个主机的span会成为模型中所说的span。


##### 2.2 植入点-Dapper的透明性
Dapper可以对应用开发者近乎零侵入的成本，对分布式控制路径进行跟踪，几乎完全依赖于基于少量通用组件库的改造。如下：
1. 当一个线程在处理跟踪控制路径的过程中，Dapper把这次跟踪的上下文在ThreadLocal中进行存储。追踪上下文是一个小而且容易复制的容器，其中承载了Scan的属性如跟踪ID和span id。
2. 当计算过程是延迟调用的或者异步的，大多数Google开发者通过线程池或者其他执行器，使用一个通用的控制流库来回调。Dapper确保所有的回调可以存储这次跟踪的上下文，而当回调函数被处罚时，这次跟踪的上下文会与适当的线程关联上。在这种方式上，Dapper可以使用trace ID和span ID来辅助构建异步调用的路径。
3. 几乎所有的Google进程间通信是建立在一个用C++和Java开发的RPC框架上。我们把跟踪植入该框架来定义RPC中所有的span。span ID和跟踪ID会从客户端发送到服务端。像那样的基于RPC的系统被广泛应用在Google中，这是一个重要的植入点。当那些非RPC通信框架发展成熟并找到了自己的用户群之后，我们会计划对RPC通信框架植入。

Dapper的跟踪数据是独立于语言的，很多在生产环境中的跟踪结合了用C++和Java写的进程数据。在后面的章节中，我们讨论应用程序的透明度时，我们会把这些理论是如何实践的进行讨论。

##### 2.3 标注：Annotation
```language
// C++
const string& request = ...;
if (HitCache())
    TRACEPRINTF("cache hit for %s", request.c_str());
else
    TRACEPRINTF("cache miss for %s", request.c_str());


// Java:
Tracer t = Tracer.getCurrentTracer();
string request = ...;
if (hitCache())
    t.Record("cache hit for "+ request);
else 
    t.Record("cache miss for "+ request);
```

上述植入点足够推导出复杂的分布式系统的跟踪细节，使得Dapper的核心功能在不改动Google应用的情况下可用。然而，Dapper还允许应用程序开发人员在Dapper跟踪的过程中添加额外的信息，已监控更高级别的系统行为，或帮助调试问题。我们允许用户通过一个简单的API定义带时间戳的Annotation，核心的示例代码如图4所示。这些Annotation可以添加任意内容。为了保护Dapper的用户过分对日志记录特别感兴趣，每个跟踪span有一个可配置的总Annotation量的上限。但是，应用程序的Annotation是不能代替用于表示span结构的信息和记录着RPC相关的信息

##### 2.4 采样率
低损耗是Dapper的一个关键设计目标。具体做法：*为了减少分布式跟踪系统对应用性能的损耗影响，那就是在遇到大量请求时只记录其中的一小部分。*

##### 2.5 跟踪的收集
![Dapper日志收集流程](bigtable.jpg)
Dapper的跟踪记录和收集管道的过程分为三个阶段（见上图）。
1. span数据写入(1)本地日志文件中。
2. Dapper的守护进程和收集组件把这些数据从生产环境的主机中拉出来(2)
3. 写到(3)Dapper的Bigtable仓库中。

一次跟踪被设计成Bigtable中的一行，每一列相当于一个span。且Bigtable支持稀疏矩阵