

在本文中，分为以下几个部分：   
第一部分：Flink 中的核心概念和基础篇，包含了 Flink 的整体介绍、核心概念、算子等考察点。  
第二部分：Flink 进阶篇，包含了 Flink 中的数据传输、容错机制、序列化、数据热点、反压等实际生产环境中遇到的问题等考察点.   
第三部分：Flink 源码篇，包含了 Flink 的核心代码实现、Job 提交流程、数据交换、分布式快照机制、Flink SQL 的原理等考察点。   

# 第一部分：Flink 中的核心概念和基础考察
## 一、 简单介绍一下 Flink
Flink 是一个框架和分布式处理引擎，用于对无界和有界数据流进行有状态计算。并且 Flink 提供了数据分布、容错机制以及资源管理等核心功能。
Flink提供了诸多高抽象层的API以便用户编写分布式任务：
DataSet API， 对静态数据进行批处理操作，将静态数据抽象成分布式的数据集，用户可以方便地使用Flink提供的各种操作符对分布式数据集进行处理，支持Java、Scala和Python。

DataStream API，对数据流进行流处理操作，将流式的数据抽象成分布式的数据流，用户可以方便地对分布式数据流进行各种操作，支持Java和Scala。

Table API，对结构化数据进行查询操作，将结构化数据抽象成关系表，并通过类SQL的DSL对关系表进行各种查询操作，支持Java和Scala。

此外，Flink 还针对特定的应用领域提供了领域库，例如： Flink ML，Flink 的机器学习库，提供了机器学习Pipelines API并实现了多种机器学习算法。 Gelly，Flink 的图计算库，提供了图计算的相关API及多种图计算算法实现。

根据官网的介绍，Flink 的特性包含：   
>支持高吞吐、低延迟、高性能的流处理   
支持带有事件时间的窗口 （Window） 操作   
支持有状态计算的 Exactly-once 语义   
支持高度灵活的窗口 （Window） 操作，支持基于 time、count、session 以及 data-driven 的窗口操作   
支持具有 Backpressure 功能的持续流模型   
支持基于轻量级分布式快照（Snapshot）实现的容错   
一个运行时同时支持 Batch on Streaming 处理和 Streaming 处理   
Flink 在 JVM 内部实现了自己的内存管理   
支持迭代计算   
支持程序自动优化：避免特定情况下 Shuffle、排序等昂贵操作，中间结果有必要进行缓存   

#二、 Flink 相比传统的 Spark Streaming 有什么区别?   
这个问题是一个非常宏观的问题，因为两个框架的不同点非常之多。但是在面试时有非常重要的一点一定要回答出来：Flink 是标准的实时处理引擎，基于事件驱动。而 Spark Streaming 是微批（Micro-Batch）的模型。
下面我们就分几个方面介绍两个框架的主要区别：
##1. 架构模型
Spark Streaming 在运行时的主要角色包括：Master、Worker、Driver、Executor，Flink 在运行时主要包含：Jobmanager、Taskmanager和Slot。
##2. 任务调度
Spark Streaming 连续不断的生成微小的数据批次，构建有向无环图DAG，Spark Streaming 会依次创建 DStreamGraph、JobGenerator、JobScheduler。
Flink 根据用户提交的代码生成 StreamGraph，经过优化生成 JobGraph，然后提交给 JobManager进行处理，JobManager 会根据 JobGraph 生成 ExecutionGraph，ExecutionGraph 是 Flink 调度最核心的数据结构，JobManager 根据 ExecutionGraph 对 Job 进行调度。
##3. 时间机制
Spark Streaming 支持的时间机制有限，只支持处理时间。 Flink 支持了流处理程序在时间上的三个定义：处理时间、事件时间、注入时间。同时也支持 watermark 机制来处理滞后数据。
##4. 容错机制
对于 Spark Streaming 任务，我们可以设置 checkpoint，然后假如发生故障并重启，我们可以从上次 checkpoint 之处恢复，但是这个行为只能使得数据不丢失，可能会重复处理，不能做到恰好一次处理语义。
Flink 则使用两阶段提交协议来解决这个问题。
#三、 Flink 的组件栈有哪些？
根据 Flink 官网描述，Flink 是一个分层架构的系统，每一层所包含的组件都提供了特定的抽象，用来服务于上层组件。
![p1](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2OKoDp7OuQQE0fccQQrNYXMcQyfq7HzfuHWQBXAt6RZn9Vz7zpFhYNdZxd4WkwicJictWlCBJHe1JOQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
自下而上，每一层分别代表：Deploy 层：该层主要涉及了Flink的部署模式，在上图中我们可以看出，Flink 支持包括local、Standalone、Cluster、Cloud等多种部署模式。Runtime 层：Runtime层提供了支持 Flink 计算的核心实现，比如：支持分布式 Stream 处理、JobGraph到ExecutionGraph的映射、调度等等，为上层API层提供基础服务。API层：API 层主要实现了面向流（Stream）处理和批（Batch）处理API，其中面向流处理对应DataStream API，面向批处理对应DataSet API，后续版本，Flink有计划将DataStream和DataSet API进行统一。Libraries层：该层称为Flink应用框架层，根据API层的划分，在API层之上构建的满足特定应用的实现计算框架，也分别对应于面向流处理和面向批处理两类。面向流处理支持：CEP（复杂事件处理）、基于SQL-like的操作（基于Table的关系操作）；面向批处理支持：FlinkML（机器学习库）、Gelly（图处理）。


----------
Orgin:  https://mp.weixin.qq.com/s?src=11&timestamp=1609689556&ver=2806&signature=FcV75ZX1wfmxokDnfOwc-Ermkji*mCgl-9WAlgsbo8-ElkTWlbhpd7-8lPikCua7FL-WXYMF*Y7yl*NIqidkbfpRqE*m0qUSdOeOhmiWB5SMiM6mtQr*sd0a3pLoOVRB&new=1
待整理完