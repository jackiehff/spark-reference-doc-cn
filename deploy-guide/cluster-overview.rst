.. _cluster_overview:

#################
集群模式概述
#################

本文简要描述了如何在集群环境中运行 Spark,  以便更容易理解其中涉及到的各个组件。如果想了解如何在集群中启动 Spark 应用程序，请参考 Spark应用程序提交指南 这篇文章。

组件
----------------------------------


Spark 应用程序在集群上作为独立的进程集合运行, 这些进程之间通过主程序（也被称为 Driver 程序）中的 SparkContext 对象来进行协调。

具体来说，为了能够在集群上运行应用程序, SparkContext 要能连接到多种类型的集群管理器（Spark 自带的 Standalone 集群管理器, Mesos 或 YARN）,  这些集群管理器用于在应用程序之间分配资源。一旦连接上集群管理器，Spark 会在集群中的各个节点上为应用程序申请 Executor 用于执行计算任务和存储数据。接着，Spark 将应用程序代码（传递给 SparkContext 的 JAR 包或者 Python 文件）发送给 Executor。最后 SparkContext 将 Task 分发给各个 Executor 执行。

.. image:: imgs/cluster-overview.png
  :scale: 90 %
  :align: center
  :target: http://spark.apache.org/docs/latest/cluster-overview.html


这个架构中有几个值得注意的地方：

1. 每个 Spark 应用程序都有自己的 Executor 进程，整个应用程序生命周期内 Executor 进程都保持着运行状态，并且以多线程方式运行所接收到的任务。这样的好处是，可以隔离各个 Spark 应用程序，从调度角度来看，每个 Driver 可以独立调度本应用程序内部的任务，从 Executor 角度来看，来自不同 Spark 应用程序的任务将会在不同的 JVM 中运行。然而这也意味着无法在多个 Spark 应用程序（SparkContext 实例）之间共享数据，除非把数据写到外部存储系统中。
2. Spark 对底层的集群管理器一无所知。只要 Spark 能获取到执行器进程，并且能与之通信即可。这样即使在一个支持多种应用程序的集群管理器（如：Mesos 或 YARN）上运行 Spark 程序也相对比较容易。
3. Driver 程序在整个生命周期内必须监听并接受其对应的各个 Executor 的连接请求（参考：spark.driver.port and spark.fileserver.port in the network config section）。因此，Driver 程序必须能够被所有 Worker 节点访问到。
4. 因为集群上的任务是由 Driver 来调度的，所以 Driver 应该在 Worker 节点附近运行，最好在同一个本地局域网内。如果你需要远程对集群发起请求，最好是开启到 Driver 的 RPC 服务并且让其就近提交操作，而不是在离集群 Worker 节点很远的地方运行 Driver。


集群管理器类型
-----------------------------------

Spark 目前支持以下3种集群管理器：

* Standalone – Spark 自带的一个简单的集群管理器，它使得启动一个 Spark 集群变得非常简单。
* Apache Mesos – 一个可以运行 Hadoop MapReduce 和服务型应用程序的通用集群管理器。
* Hadoop YARN – Hadoop 2 的资源管理器。
* Kubernetes (experimental) – 除上述之外，还有Kubernetes的实验支持。 Kubernetes是一个提供以容器为中心的基础设施的开源平台。 Kubernetes的支持正在积极开发在apache-spark-on-k8s Github组织中。 有关文档，请参阅该项目的自述文件.

提交 Spark 应用
----------------------------------

利用 spark-submit 脚本，可以向 Spark 所支持的任意一种集群管理器提交应用程序。更多详细信息请参见 Spark应用程序提交指南 这篇文章。

监控
---------------------------------

每个 Driver 程序都有一个 web UI，通常会绑定到 4040 端口，它会展示集群上正在执行的 Tasks、Executors 以及存储空间使用等信息。你只需要在浏览器中输入 http://<driver-node>:4040 这个地址即可访问这个 web UI。监控指南 这篇文章中描述了其它的一些监控选项。

作业调度
--------------------------------

Spark 可以在应用程序之间（集群管理器这一层面）和之内（如：同一个SparkContext对象运行了多个计算作业）控制资源分配。更多详细信息参见 作业调度概览 这篇文章。


术语表
---------------------------

下表总结了一些之前看到的集群概念相关的术语：


============================    ====================================================================
术语                                  含义
============================    ====================================================================
Application(应用程序)            构建于 Spark 上的用户程序。它由集群上的一个驱动器（driver）和多个执行器（executor）组成。
Application jar(应用程序jar)     包含 Spark 应用程序的 jar 包。有时候，用户会想要把应用程序代码及其依赖打到一起，形成一个 “uber jar”（包含自身以及所有依赖库的jar包），注意这时候不要把 Spark 或 Hadoop 的库打进来，这些库会在运行时加载。
Driver Program(驱动器程序)        运行 main 函数并创建 SparkContext 的进程。
Cluster Manager(集群管理器)       用于在集群上分配资源的外部服务（如：Standalone 集群管理器、Mesos或者YARN。
Deploy Mode(部署模式)             用于区分 Driver 进程在哪儿运行。在 "cluster" 模式下，框架在集群内部启动 Driver 程序；在 "client" 模式下，提交者在集群外部启动 Driver 程序。
Worker Node(工作节点)             集群中可以运行应用程序代码的任意一个节点。
Executor(执行器)                 在集群 Worker 节点上为某个应用程序启动的工作进程, 它专门用于运行计算任务，并在内存或磁盘上保存数据。每个应用程序都有自己的 Executor。
Task(任务)                       下发给 Executor 的工作单元。
Job(作业)                        一个并行计算 Job 由一组 Task 组成，并由 Spark action（如：save、collect）触发启动；你将会在 Driver 的日志中看到这个术语。
Stage(步骤)                      每个 Job 可以划分为多个更小的 Task 集合，这些 Task 集合称为 Stage，这些 Stage 彼此依赖形成一个有向无环图（类似于 MapReduce 中的 map 和 reduce）；你将会在 Driver 的日志中看到这个术语。
============================    ====================================================================

