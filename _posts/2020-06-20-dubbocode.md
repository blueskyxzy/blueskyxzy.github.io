---
layout: post
title: dubbo源码研究
date: 2020-06-20
tags: 技术    
---

### 简介
分布式服务框架

    高性能和透明化的RPC远程服务调用方案
    SOA服务治理方案
    Apache MINA 框架基于Reactor模型通信框架，基于tcp长连接

### 网上提供的原理
采用单一长连接和NIO异步通讯，适合于小数据量大并发的服务调用，以及服务消费者机器数远大于服务提供者机器数的情况  

作为RPC：支持各种传输协议，如dubbo,hession,json,fastjson，底层采用mina,netty长连接进行传输！典型的provider和consumer模式  
作为SOA：具有服务治理功能，提供服务的注册和发现！用zookeeper实现注册中心！  
启动时候服务端会把所有接口注册到注册中心，并且订阅configurators,服务消费端订阅provide，configurators,routers,订阅变更时，zk会推送providers,configuators，routers,启动时注册长连接，进行通讯！  
proveider和provider启动后，后台启动定时器，发送统计数据到monitor（监控中心）！提供各种容错机制和负载均衡策略！！  

 ![dubbo](/images/posts/dubbo/dubbo.jpg)  
 
    1.client一个线程调用远程接口，生成一个唯一的ID（比如一段随机字符串，UUID等），Dubbo是使用AtomicLong从0开始累计数字的
    2.将打包的方法调用信息（如调用的接口名称，方法名称，参数值列表等），和处理结果的回调对象callback，全部封装在一起，组成一个对象object
    3.向专门存放调用信息的全局ConcurrentHashMap里面put(ID, object)
    4.将ID和打包的方法调用信息封装成一对象connRequest，使用IoSession.write(connRequest)异步发送出去
    5.当前线程再使用callback的get()方法试图获取远程返回的结果，在get()内部，则使用synchronized获取回调对象callback的锁， 再先检测是否已经获取到结果，如果没有，然后调用callback的wait()方法，释放callback上的锁，让当前线程处于等待状态。
    6.服务端接收到请求并处理后，将结果（此结果中包含了前面的ID，即回传）发送给客户端，客户端socket连接上专门监听消息的线程收到消息，分析结果，取到ID，再从前面的ConcurrentHashMap里面get(ID)，从而找到callback，将方法调用结果设置到callback对象里。
    7.监听线程接着使用synchronized获取回调对象callback的锁（因为前面调用过wait()，那个线程已释放callback的锁了），再notifyAll()，唤醒前面处于等待状态的线程继续执行（callback的get()方法继续执行就能拿到调用结果了），至此，整个过程结束

两种Invoker：服务提供Invoker和服务消费Invoker

Consumer服务消费者，Provider服务提供者。Container服务容器。消费当然是invoke提供者了，invoke这条实线按照图上的说明当然同步的意思了，多说一句，在实际调用过程中，Provider的位置对于Consumer来说是透明的，上一次调用服务的位置（IP地址）和下一次调用服务的位置，是不确定的。这个地方就是实现了软负载。

服务提供者先启动start，然后注册register服务。

消费订阅subscribe服务，如果没有订阅到自己想获得的服务，它会不断的尝试订阅。新的服务注册到注册中心以后，注册中心会将这些服务通过notify到消费者。

Monitor这是一个监控，图中虚线表明Consumer 和Provider通过异步的方式发送消息至Monitor，Consumer和Provider会将信息存放在本地磁盘，平均1min会发送一次信息。

Monitor在整个架构中是可选的（图中的虚线并不是可选的意思），Monitor功能需要单独配置，不配置或者配置以后，Monitor挂掉并不会影响服务的调用。

### Java中SPI机制
服务发现机制 就是一个“基于接口的编程＋策略模式＋配置文件”组合实现的动态加载机制。  
为某个接口寻找服务实现的机制。有点类似IOC的思想，就是将装配的控制权移到程序之外。解耦合，延迟加载  
ServiceLoader 来加载配置文件中指定的实现，ServiceLoader可以跨越jar包获取META-INF下的配置文件，通过反射方法Class.forName()加载类对象，instance()方法将类实例化。把实例化后的类缓存到providers对象中  
SPI的方式使得源框架，不必关心接口的实现类的路径。如import 导入实现类，反射Class.forName("com.mysql.jdbc.Driver")

dubbo中也有用到，Dubbo 并未使用 Java 原生的 SPI 机制，而是对其进行了增强。Dubbo SPI 是通过键值对的方式进行配置，这样我们可以按需加载指定的实现类  
很多拓展都是通过 SPI 机制进行加载的，比如 Protocol、Cluster、LoadBalance.  
dubbo自适应拓展机制,并不想在框架启动阶段被加载，而是希望在拓展方法被调用时，根据运行时参数进行加载。首先 Dubbo 会为拓展接口生成具有代理功能的代码。然后通过 javassist 或 jdk 编译这段代码，得到 Class 类。最后再通过反射创建代理类，  
表现就是传统的代理逻辑不同，所代理的对象是在 makeWheel 方法中通过 SPI 加载得到

这种机制类似于Spring中也有一种类似与Java SPI的加载机制Starter

RPC 协议扩展，封装远程调用细节。   
org.apache.dubbo.rpc.Protocol  
org.apache.dubbo.rpc.Exporter  
org.apache.dubbo.rpc.Invoker  
  
### 官网给的架构

 ![dubbo framework](/images/posts/dubbo/dubbo-framework.jpg)  
 
 各层说明：
* config 配置层：对外配置接口，以 ServiceConfig, ReferenceConfig 为中心，可以直接初始化配置类，也可以通过 spring 解析配置生成配置类
* proxy 服务代理层：服务接口透明代理，生成服务的客户端 Stub 和服务器端 Skeleton, 以 ServiceProxy 为中心，扩展接口为 ProxyFactory
* registry 注册中心层：封装服务地址的注册与发现，以服务 URL 为中心，扩展接口为 RegistryFactory, Registry, RegistryService
* cluster 路由层：封装多个提供者的路由及负载均衡，并桥接注册中心，以 Invoker 为中心，扩展接口为 Cluster, Directory, Router, LoadBalance
* monitor 监控层：RPC 调用次数和调用时间监控，以 Statistics 为中心，扩展接口为 MonitorFactory, Monitor, MonitorService
* protocol 远程调用层：封装 RPC 调用，以 Invocation, Result 为中心，扩展接口为 Protocol, Invoker, Exporter
* exchange 信息交换层：封装请求响应模式，同步转异步，以 Request, Response 为中心，扩展接口为 Exchanger, ExchangeChannel, ExchangeClient, ExchangeServer
* transport 网络传输层：抽象 mina 和 netty 为统一接口，以 Message 为中心，扩展接口为 Channel, Transporter, Client, Server, Codec
* serialize 数据序列化层：可复用的一些工具，扩展接口为 Serialization, ObjectInput, ObjectOutput, ThreadPool

注：

* 在 RPC 中，Protocol 是核心层，也就是只要有 Protocol + Invoker + Exporter 就可以完成非透明的 RPC 调用，然后在 Invoker 的主过程上 Filter 拦截点。
* 图中的 Consumer 和 Provider 是抽象概念，只是想让看图者更直观的了解哪些类分属于客户端与服务器端，不用 Client 和 Server 的原因是 Dubbo 在很多场景下都使用 Provider, Consumer, Registry, Monitor 划分逻辑拓普节点，保持统一概念。
* 而 Cluster 是外围概念，所以 Cluster 的目的是将多个 Invoker 伪装成一个 Invoker，这样其它人只要关注 Protocol 层 Invoker 即可，加上 Cluster 或者去掉 Cluster 对其它层都不会造成影响，因为只有一个提供者时，是不需要 Cluster 的。
* Proxy 层封装了所有接口的透明化代理，而在其它层都以 Invoker 为中心，只有到了暴露给用户使用时，才用 Proxy 将 Invoker 转成接口，或将接口实现转成 Invoker，也就是去掉 Proxy 层 RPC 是可以 Run 的，只是不那么透明，不那么看起来像调本地服务一样调远程服务。
* 而 Remoting 实现是 Dubbo 协议的实现，如果你选择 RMI 协议，整个 Remoting 都不会用上，Remoting 内部再划为 Transport 传输层和 Exchange 信息交换层，Transport 层只负责单向消息传输，是对 Mina, Netty, Grizzly 的抽象，它也可以扩展 UDP 传输，而 Exchange 层是在传输层之上封装了 Request-Response 语义。
* Registry 和 Monitor 实际上不算一层，而是一个独立的节点，只是为了全局概览，用层的方式画在一起。


模块说明：

* dubbo-common 公共逻辑模块：包括 Util 类和通用模型。
* dubbo-remoting 远程通讯模块：相当于 Dubbo 协议的实现，如果 RPC 用 RMI协议则不需要使用此包。
* dubbo-rpc 远程调用模块：抽象各种协议，以及动态代理，只包含一对一的调用，不关心集群的管理。
* dubbo-cluster 集群模块：将多个服务提供方伪装为一个提供方，包括：负载均衡, 容错，路由等，集群的地址列表可以是静态配置的，也可以是由注册中心下发。
* dubbo-registry 注册中心模块：基于注册中心下发地址的集群方式，以及对各种注册中心的抽象。
* dubbo-monitor 监控模块：统计服务调用次数，调用时间的，调用链跟踪的服务。
* dubbo-config 配置模块：是 Dubbo 对外的 API，用户通过 Config 使用Dubbo，隐藏 Dubbo 所有细节。
* dubbo-container 容器模块：是一个 Standlone 的容器，以简单的 Main 加载 Spring 启动，因为服务通常不需要 Tomcat/JBoss 等 Web 容器的特性，没必要用 Web 容器去加载服务。

Dubbo 服务调用过程   

 ![dubbo framework](/images/posts/dubbo/send-request-process.jpg)  
 
首先服务消费者通过代理对象 Proxy 发起远程调用，接着通过网络客户端 Client 将编码后的请求发送给服务提供方的网络层上，也就是 Server。  
Server 在收到请求后，首先要做的事情是对数据包进行解码。然后将解码后的请求发送至分发器 Dispatcher，再由分发器将请求派发到指定的线程池上，  
最后由线程池调用具体的服务。这就是一个远程调用请求的发送与接收过程。至于响应的发送与接收过程，这张图中没有表现出来

### 源码
网上主要是总结性的流程，知道个概念，要看明白还是要靠看源码

dubbo源码的知识点总结：  
集群容错， 服务发布原理，服务引用， 服务暴露
Directory， Invoker， router，LoadBalance，cluster，Exporter

Invoker  
具有远程调用功能的对象  
Invoker是从代理抽象处理的调用接口，可由代理工厂接口ProxyFactory的实现创建，如javassistProxyFactory,这里用Javassist动态字节码技术做代理  
在这里分消费者Invoker,生产者Invoker等  


router路由  
Router应用场景有：应用隔离,读写分离,灰度发布。  
灰度发布就是不停止对外的服务,也就是让用户感觉不到你在发布。让一部分人先用新版本的服务   
路由就是管理地址，IP等，如禁用部分IP等控制，哪个服务器可以调哪个地址等  
Dubbo 目前提供了三种服务路由实现，分别为条件路由 ConditionRouter、脚本路由 ScriptRouter 和标签路由 TagRouter  

Cluster集群接口   
将 Directory 中的多个 Invoker 伪装成一个 Invoker，对上层透明，伪装过程包含了容错逻辑，调用失败后，重试另一个  
应对出错情况采取的策略  
集群 Cluster 用途是将多个服务提供者合并为一个 Cluster Invoker，并将这个 Invoker 暴露给服务消费者。   
这样一来，服务消费者只需通过这个 Invoker 进行远程调用即可，至于具体调用哪个服务提供者，以及调用失败后如何处理等问题，现在都交给集群模块去处理。  
集群实现，包含但不限于 Failover Cluster、Failfast Cluster 和 Failsafe Cluster 等

directory服务目录  
。服务目录中存储了一些和服务提供者有关的信息，通过服务目录，服务消费者可获取到服务提供者的信息，比如 ip、端口、服务协议等。通过这些信息，服务消费者就可通过 Netty 等客户端进行远程调用  
directory接口主要有两个实现类,一个是StaticDirectory,一个是RegistryDirectory  
RegistryDirectory 动态管理List<Invoker>，比如注册中心推送变更   
Directory获取invoker是从methodInvokerMap中获取的，这些主要都是读操作，它的写操作在回调方法notify的时候操作的  
RegistryDirectory实现了 NotifyListener 接口  ，RegistryDirectory 中有几个比较重要的逻辑，第一是 Invoker 的列举逻辑，第二是接收服务配置变更的逻辑，第三是 Invoker 列表的刷新逻辑   

。服务目录中存储了一些和服务提供者有关的信息，通过服务目录，服务消费者可获取到服务提供者的信息，比如 ip、端口、服务协议等。通过这些信息，服务消费者就可通过 Netty 等客户端进行远程调用  
实际上服务目录在获取注册中心的服务配置信息后，会为每条配置信息生成一个 Invoker 对象，并把这个 Invoker 对象存储起来，这个 Invoker 才是服务目录最终持有的对象  
服务目录可以看成Invoker 集合，且这个集合中的元素会随注册中心的变化而进行动态调整。 

LoadBalance(负载均衡)   
Dubbo 提供了4种负载均衡实现，分别是基于权重随机算法的 RandomLoadBalance、基于最少活跃调用数算法的 LeastActiveLoadBalance、基于 hash 一致性的 ConsistentHashLoadBalance，以及基于加权轮询算法的 RoundRobinLoadBalance  
AbstractLoadBalance的负载均衡的入口方法 select  
另外还有 服务提供者权重计算逻辑，该过程主要用于保证当服务运行时长小于服务预热时间时，对服务进行降权，避免让服务在启动之初就处于高负载状态。服务预热是一个优化手段，与此类似的还有 JVM 预热。主要目的是让服务启动后“低功率”运行一段时间，使其效率慢慢提升至最佳状态。
RandomLoadBalance 是加权随机算法   就是权重分批随机数区间，取随机数区间来分配
LeastActiveLoadBalance  最小活跃数负载均衡  每个服务提供者对应一个活跃数 active。初始情况下，所有服务提供者活跃数均为0。每收到一个请求，活跃数加1，完成请求后则将活跃数减1。这样性能快的处理快，活跃数降的快，能分配更多的负载
ConsistentHashLoadBalance   一致性 hash 算法提出之初是用于大规模缓存系统的负载均衡 为缓存节点生成一个 hash，并将这个 hash 投射到 [0, 232 - 1] 的圆环上  目的是通过引入虚拟节点，让 Invoker 在圆环上分散开来，避免数据倾斜问题


dubbo中的服务降级分成两个  
屏蔽(mock=force)  
容错(mock=fail)  

如 MockClusterInvoker 
从no mock(正常情况),force:direct mock(屏蔽),fail-mock(容错)三种情况我们也可以看出,  
普通情况是直接调用,容错的情况是调用失败后,返回一个设置的值.而屏蔽就很暴力了,直接连调用都不调用,就直接返回一个之前设置的值.

### 总结

