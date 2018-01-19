Spark安全
=======================================

Spark 目前已经支持以共享秘钥的方式进行身份认证。开启身份认证配置参数为 spark.authenticate。这个配置参数决定了Spark通讯协议是否使用共享秘钥做身份验证。验证过程就是一个基本的握手过程，确保通讯双方都有相同的秘钥并且可以互相通信。如果共享秘钥不同，双方是不允许通信的。共享秘钥可用以下方式创建：

* 对于以 YARN 方式部署的 Spark，将 spark.authenticate 设为true可以自动生成并分发共享秘钥。每个 Spark 应用会使用唯一的共享秘钥。
* 而对于其他部署类型，需要在每个节点上设置 spark.authenticate.secret 参数。这个秘钥将会在由所有 Master/Workers以及各个Spark应用共享。


Web UI
---------------------------------------

Spark UI 也可以通过配置 spark.ui.filters 来使用 javax servlet filters 确保安全性，

认证
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

因为某些用户可能希望web UI上的某些数据应该保密，并对其他用户不可见。用户可以自定义 javax servlet filter 来对登陆用户进行认证，Spark会根据用户的ACL（访问控制列表）来确保该登陆用户有权限访问某个Spark应用的web UI。Spark ACL的行为可由 spark.acls.enable 和 spark.ui.view.acls 共同控制。注意，启动Spark应用的用户总是会有权限访问该应用对应的UI。而在YARN模式下，Spark web UI会使用YARN的web应用代理机制，通过Hadoop过滤器进行认证。

Spark还支持修改运行中的Spark应用对应的ACL以便更改其权限控制，不过这可能会导致杀死应用或者杀死任务的动作。这一特性由 spark.acls.enable 和 spark.modify.acls 共同控制。注意，如果你需要web UI的认证，比如为了能够在web UI上使用杀死应用的按钮，那么就需要将用户同时添加到modify ACL和view ACL中。在YARN上，控制用户修改权限的modify ACL是通过YARN接口传进来的。

Spark可以允许多个管理员共同管理ACL，而且这些管理员总是能够访问和修改所有的Spark应用。而管理员本身是由 spark.admin.acls 配置的。在一个共享的多用户集群中，你可能需要配置多个管理员，另外，该配置也可以用于支持开发人员调试Spark应用。


事件日志
---------------------------------------

如果你启用了事件日志（event logging），那么这些日志对应的目录（spark.eventLog.dir）需要事先手动创建并设置好权限。如果你要限制这些日志文件的权限，首先还是要将日志目录的权限设为 drwxrwxrwxt，目录owner应该是启动history server的超级用户，对应的组限制为该超级用户所在组。这样其他所有用户就只能往该目录下写日志，但同时又不能删除或改名。事件日志文件只有在用户或者用户组具有读写权限时才能写入。


加密
---------------------------------------

Spark对Akka和HTTP协议（广播和文件服务器中使用）支持SSL。同时，对数据块传输服务（block transfer service）支持SASL加密。Spark web UI暂时还不支持任何加密。

Spark的临时数据存储，如：混洗中间文件、缓存数据或其他应用相关的临时文件，目前也没有加密。如果需要加密这些数据，只能通过配置集群管理器将数据存储到加密磁盘上。

SSL配置
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Spark SSL相关配置是层级式的。用户可以配置SSL的默认配置，以便支持所有的相关通讯协议，当然，如果设置了针对某个具体协议配置值，其值将会覆盖默认配置对应的值。这种方式主要是为了方便用户，用户可以轻松地为所有协议都配置好默认值，同时又不会影响针对某一个具体协议的特殊配置需求。通用的SSL默认配置在 spark.ssl 这一配置命名空间下，对于Akka的SSL配置，在spark.ssl.akka下，而对用于广播和文件服务器中的HTTP协议，其配置在 spark.ssl.fs 下。详细的清单见配置指南（configuration page）。

SSL必须在每个节点上都配置好，并且包括各个使用特定通讯协议的相关模块。

YARN模式
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

key-store 文件可以由客户端准备好，然后作为Spark应用的一部分分发到各个执行器（executor）上使用。用户也可以在Spark应用启动前，通过配置 spark.yarn.dist.files 或者 spark.yarn.dist.archives 来部署key-store文件。这些文件传输过程的加密是由YARN本身负责的，和Spark就没什么关系了。

对于一些长期运行并且可以写HDFS的Spark应用，如：Spark Streaming 上的应用，可以用 spark-submit 的 –principal 和 –keytab 参数分别设置principal 和 keytab 信息。keytab文件将会通过 Hadoop Distributed Cache（如果YARN配置了 SSL并且HDFS启用了加密，那么分布式缓存的传输也会被加密） 复制到Application Master所在机器上。Kerberos的登陆信息将会被principal和keytab周期性地刷新，同时HDFS所需的代理token也会被周期性的刷新，这样Spark应用就能持续地写入HDFS了。

独立模式
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

独立模式下，用户需要为master和worker分别提供key-store和相关配置选项。这些配置可以通过在SPARK_MASTER_OPTS 和 SPARK_WORKER_OPTS，或者 SPARK_DEAMON_JAVA_OPTS环境变量中添加相应的java系统属性来设置。独立模式下，用户可以通过worker的SSL设置来改变执行器（executor）的配置，因为这些执行器进程都是worker的子进程。不过需要注意的是，执行器如果需要启用本地SSL配置值（如：从worker进程继承而来的环境变量），而不是用户在客户端设置的值，就需要将 spark.ssl.userNodeLocalConf 设为 true。

准备key-stores
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

key-stores 文件可以由 keytool 程序生成。keytool 相关参考文档见这里：here。独立模式下，配置key-store和trust-store至少有这么几个基本步骤：

* 为每个节点生成一个秘钥对
* 导出各节点秘钥对中的公匙（public key）到一个文件
* 将所有这些公匙导入到一个 trust-store 文件中
* 将该trust-store文件发布到所有节点

配置SASL加密
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

启用认证后（spark.authenticate），数据块传输服务（block transfer service）可以支持SASL加密。如需启用SASL加密的话，还需要在 Spark 应用中设置 spark.authenticate.enableSaslEncryption 为 true。

如果是开启了外部混洗服务（external shuffle service），那么只需要将 spark.network.sasl.serverAlwaysEncrypt 设为true即可禁止非加密的网络连接。因为这个配置一旦启用，所有未使用 SASL加密的Spark应用都无法连接到外部混洗服务上。


配置网络安全端口
---------------------------------------

Spark计算过程中大量使用网络通信，而有些环境中对网络防火墙的设置要求很严格。下表列出来Spark用于通讯的一些主要端口，以及如何配置这些端口。

仅独立部署适用
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

=============================================     ==================    ==================    ==================    =============================================     ==========================
访问源                                             访问目标                默认端口               用途                   配置                                               注意
=============================================     ==================    ==================    ==================    =============================================     ==========================
Browser                                           Standalone Master     8080                  Web UI                spark.master.ui.port/SPARK_MASTER_WEBUI_PORT      基于Jetty。仅独立模式有效。
Browser                                           Standalone Worker     8081                  Web UI                spark.worker.ui.port/SPARK_WORKER_WEBUI_PORT      基于Jetty。仅独立模式有效。
Driver/Standalone Worker                          Standalone Master     7077                  提交作业/合并集群        SPARK_MASTER_PORT                                 设为0表示随机。仅独立模式有效。
Standalone Master                                 Standalone Worker     (random)              调度执行器              SPARK_WORKER_PORT                                 设为0表示随机。仅独立模式有效。
=============================================     ==================    ==================    ==================    =============================================     ==========================


所有集群管理器适用
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

=============================================     ==================    ==================    =====================================    ========================     ========================================
访问源                                             访问目标                默认端口               用途                                      配置                          注意
=============================================     ==================    ==================    =====================================    ========================     ========================================
Browser                                           Application           4040                  Web UI                                   spark.ui.port                基于Jetty。
Browser                                           History Server        18080                 Web UI                                   spark.history.ui.port        基于Jetty。
Executor/Standalone Master                        Driver                (random)              连接Spark应用程序/Executor状态变化通知        spark.driver.port            设为0表示随机。
Executor/Driver                                   Executor/Driver       (random)              数据块管理器端口                            spark.blockManager.port      基于ServerSocketChannel使用原始socket通信。
=============================================     ==================    ==================    =====================================    ========================     ========================================

更多关于安全方面的配置参数，请参考配置指南（configuration page），安全方面的一些实现细节可以参考 org.apache.spark.SecurityManager。
