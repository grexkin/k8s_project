1. docker三大核心

```
container 
image  
registry
```

2.etcd 原理和组建

```
- 原理
在etcd的集群中会选举出一位leader，其他etcd服务节点就会成为follower，在此过程其他follower会同步leader的数据。
由于etcd集群必须能够选举出leader才能正常工作，所以部署的服务器数量必须是奇数，例如：1，3，5，7，9 的etcd节点数量。
- 组件
   HTTP Server： 用于处理用户发送的API请求以及其它etcd节点的同步与心跳信息请求。
   Store：用于处理etcd支持的各类功能的事务，包括数据索引、节点状态变更、监控与反馈、事件处理与执行等等，是etcd对用户提供的大多数API功能的具体实现。
   Raft：Raft强一致性算法的具体实现，是etcd的核心。
   WAL：Write Ahead Log（预写式日志），是etcd的数据存储方式。除了在内存中存有所有数据的状态以及节点的索引以外，etcd就通过WAL进行持久化存储。WAL中，所有的数据提交前都会事先记录日志。Snapshot是为了防止数据过多而进行的状态快照；Entry表示存储的具体日志内容
```

3.etcd 选举过程

```
Raft中使用心跳机制来出发leader选举。当服务器启动的时候，服务器成为follower。只要follower从leader或者candidate收到有效的RPCs就会保持follower状态。如果follower在一段时间内（该段时间被称为election timeout）没有收到消息，则它会假设当前没有可用的leader，然后开启选举新leader的流程。流程如下：

1. follower增加当前的term，转变为candidate。
2. candidate投票给自己，并发送RequestVote RPC给集群中的其他服务器。
3. 收到RequestVote的服务器，在同一term中只会按照先到先得投票给至多一个candidate。且只会投票给log至少和自身一样新的candidate。
4. candidate节点保持2的状态，直到下面三种情况中的一种发生。
4.1 该节点赢得选举。即收到大多数的节点的投票。则其转变为leader状态。
4.2 另一个服务器成为了leader。即收到了leader的合法心跳包（term值等于或大于当前自身term值）。则其转变为follower状态。
4.3 一段时间后依然没有胜者。该种情况下会开启新一轮的选举。
Raft中使用随机选举超时时间来保证当票数相同无法确定leader的问题可以被快速的解决。

```

4.etcd协议用的是什么

```
etcd是用Go编写的，它具有出色的跨平台支持，小型二进制文件和背后的优秀社区。etcd机器之间的通信通过Raft一致性算法处理。
```

5.k8s 怎么升级

```
高可用集群
1. 选择业务低峰期
2. LB摘除负载较低的master
3. 在所有master节点进行升级，升级管理断服务kube-controller-manager,kube-apiserver,kube-scheduler,kube-proxy
4. 所有master节点都安装新版本的kubeadm  （ kubeadm upgrade plan 查看升级计划）
5. 更新master节点的kubectl,kubelet
6. 重启kubelet服务
7. 验证master升级状态
8. 更新node节点kubeadm 
9. 验证集群状态
10. 验证服务状态
```

6.k8s 架构各个组件作用

```
- master： 主节点
    - kube-apiserver： https://k8smeetup.github.io/docs/admin/kube-apiserver/
        -  为api对象验证并配置数据，包括pods，services，APIserver提供Restful操作和到集群共享状态的前端，所有其它组件通过apiserver进行交互
    - kube-scheduler：https://k8smeetup.github.io/docs/admin/kube-scheduler/
        - 具有丰富的资源策略，能够感知拓扑变化，支持特定负载的功能组件，它对集群的可用性，性能表现以及容量都影响巨大，scheduler需要考虑独立的和集体的资源需求，服务质量需求，硬件、软件，策略限制，亲和与反亲和规范，数据位置吗内部负载接口，截止时间等等，如有特定的负载需求可以通过apiserver暴露出来
    - kube-controller-manager: https://k8smeetup.github.io/docs/admin/kube-controller-manager/
        - 作为集群内部的控制中心，负责集群内部的Node，Pod副本，服务端点，命名空间，服务账号，资源配额的管理，当某个Node意外宕机时，controller-manager会及时发现并执行自动修复，确保集群始终处于预期的工作状态

- Node节点
    - kube-proxy： https://k8smeetup.github.io/docs/admin/kube-proxy/
        - 维护node节点上的网络规则，实现用户访问请求的转发，其实就是转发给service，需要管理员指定service和NodePort的对应关系

        - Kubernetes 网络代理运行在 node 上。它反映了 node 上 Kubernetes API 中定义的服务，并可以通过一组后端进行简单的 TCP、UDP 流转发或循环模式（round robin)）的 TCP、UDP 转发。目前，服务的集群 IP 和端口是通过 Docker-links 兼容的环境变量发现的，这些环境变量指定了服务代码打开的端口。有一个可选的 addon 为这些集群 IP 提供集群 DNS。用户必须使用 apiserver API 创建一个服务来配置代理。

    - kubelet： https://k8smeetup.github.io/docs/admin/kubelet/
        - kubelet 是运行在每个节点上的主要的“节点代理”，它按照 PodSpec 中的描述工作。 PodSpec 是用来描述一个 pod 的 YAML 或者 JSON 对象。kubelet 通过各种机制（主要通过 apiserver ）获取一组 PodSpec 并保证在这些 PodSpec 中描述的容器健康运行。kubelet 不管理不是由 Kubernetes 创建的容器。
        - 除了来自 apiserver 的 PodSpec ，还有 3 种方式可以将容器清单提供给 kubelet 。
        - 文件：在命令行指定的一个路径，在这个路径下的文件将被周期性的监视更新，默认监视周期是 20 秒并可以通过参数配置。
        - HTTP端点：在命令行指定的一个HTTP端点，该端点每 20 秒被检查一次并且可以通过参数配置检查周期。
        - HTTP服务：kubelet 还可以监听 HTTP 服务并响应一个简单的 API 来创建一个新的清单
    
- ETCD： 存储所有集群数据
组件官网参考： https://kubernetes.io/zh/docs/concepts/overview/components/
```

7.k8s pod 创建流程

```
1. kubectl 向 k8s api server 发起一个create pod 请求(即我们使用Kubectl敲一个create pod命令) 。
2. k8s api server接收到pod创建请求后，不会去直接创建pod；而是生成一个包含创建信息的yaml。
3. apiserver 将刚才的yaml信息写入etcd数据库。到此为止仅仅是在etcd中添加了一条记录， 还没有任何的实质性进展。
4. scheduler 查看 k8s api ，类似于通知机制。
首先判断：pod.spec.Node == null?
若为null，表示这个Pod请求是新来的，需要创建；因此先进行调度计算，找到最“闲”的node。
然后将信息在etcd数据库中更新分配结果：pod.spec.Node = nodeA (设置一个具体的节点)
ps:同样上述操作的各种信息也要写到etcd数据库中中。
5. kubelet 通过监测etcd数据库(即不停地看etcd中的记录)，发现 k8s api server 中有了个新的Node；
如果这条记录中的Node与自己的编号相同(即这个Pod由scheduler分配给自己了)；
则调用node中的docker api，创建container。
```

8.calico 网络原理

```
Calico把每个操作系统的协议栈认为是一个路由器，然后把所有的容器认为是连在这个路由器上的网络终端，在路由器之间跑标准的路由协议——BGP的协议，然后让它们自己去学习这个网络拓扑该如何转发。所以Calico方案其实是一个纯三层的方案，也就是说让每台机器的协议栈的三层去确保两个容器，跨主机容器之间的三层连通性。

对于控制平面，它每个节点上会运行两个主要的程序，一个是Felix，它会监听ECTD中心的存储，从它获取事件，比如说用户在这台机器上加了一个IP，或者是分配了一个容器等。接着会在这台机器上创建出一个容器，并将其网卡、IP、MAC都设置好，然后在内核的路由表里面写一条，注明这个IP应该到这张网卡。绿色部分是一个标准的路由程序，它会从内核里面获取哪一些IP的路由发生了变化，然后通过标准BGP的路由协议扩散到整个其他的宿主机上，让外界都知道这个IP在这里，你们路由的时候得到这里来。

由于Calico是一种纯三层的实现，因此可以避免与二层方案相关的数据包封装的操作，中间没有任何的NAT，没有任何的overlay，所以它的转发效率可能是所有方案中最高的，因为它的包直接走原生TCP/IP的协议栈，它的隔离也因为这个栈而变得好做。因为TCP/IP的协议栈提供了一整套的防火墙的规则，所以它可以通过IPTABLES的规则达到比较复杂的隔离逻辑。
```

9.ceph 各个组件和原理

```
## 组件
1. Monitors是一个Ceph集群需要多个Monitor组成的小集群，它们通过Paxos同步数据，用来保存OSD的元数据。
2. OSD全称Object Storage Device，也就是负责响应客户端请求返回具体数据的进程。一个Ceph集群一般都有很多个OSD。
3. MDS全称Ceph Metadata Server，是CephFS服务依赖的元数据服务。
4. Ceph最底层的存储单元是Object对象，每个Object包含元数据和原始数据。
5. PG全称Placement Grouops，是一个逻辑的概念，一个PG包含多个OSD。引入PG这一层其实是为了更好的分配数据和定位数据。
6. RADOS全称Reliable Autonomic Distributed Object Store，是Ceph集群的精华，用户实现数据分配、Failover等集群操作。
7. Librados是Rados提供库，因为RADOS是协议很难直接访问，因此上层的RBD、RGW和CephFS都是通过librados访问的，目前提供PHP、Ruby、Java、Python、C和C++支持。
8. CRUSH是Ceph使用的数据分布算法，类似一致性哈希，让数据分配到预期的地方。
9. RBD全称RADOS block device，是Ceph对外提供的块设备服务。
10. RGW全称RADOS gateway，是Ceph对外提供的对象存储服务，接口与S3和Swift兼容。
11. CephFS全称Ceph File System，是Ceph对外提供的文件系统服务

```

10.schedule 机制

```
scheduler的功能不多，但逻辑比较复杂，里面有很多考虑的因素，总结下来大致有如下几点：
1. Leader选主，确保集群中只有一个scheduler在工作，其它只是高可用备份实例。通过endpoint:kube-scheduler作为仲裁资源。
2. Node筛选，根据设置的条件、资源要求等，匹配出所有满足分配的Node结点。
3. 最优Node选择。在所有满足条件的Node中，根据定义好的规则来打分，取分数最高的。如果有相同分数的，则采用轮询方式。
4. 为了响应高优先级的资源分配，增加了抢占功能。scheduler有权删除一些低优先级的Pod，以释放资源给高优先级的Pod来使用。
```

11.prometheus-operator 机制和原理

```
作为一个控制器，他会去创建Prometheus、ServiceMonitor、AlertManager以及PrometheusRule4个CRD资源对象，然后会一直监控并维持这4个资源对象的状态。
1. 其中创建的prometheus这种资源对象就是作为Prometheus Server存在，而ServiceMonitor就是exporter的各种抽象，exporter前面我们已经学习了，是用来提供专门提供metrics数据接口的工具，Prometheus就是通过ServiceMonitor提供的metrics数据接口去 pull 数据的，当然alertmanager这种资源对象就是对应的AlertManager的抽象，而PrometheusRule是用来被Prometheus实例使用的报警规则文件。
2. 这样我们要在集群中监控什么数据，就变成了直接去操作 Kubernetes 集群的资源对象了，是不是方便很多了。上图中的 Service 和 ServiceMonitor 都是 Kubernetes 的资源，一个 ServiceMonitor 可以通过 labelSelector 的方式去匹配一类 Service，Prometheus 也可以通过 labelSelector 去匹配多个ServiceMonitor。
```

12.sts和deploymnet 区别

```
- sts（statefulSet）： 有状态应用控制器
- deployment: 无状态应用控制器
```

13.crd 原理作用

```
CRD是自定义资源类型的操作
```

14. etcd数据同步原理
```
日志复制
当前 Leader 收到客户端的日志（事务请求）后先把该日志追加到本地的 Log 中，然后通过 heartbeat 把该 Entry 同步给其他 Follower，Follower 接收到日志后记录日志然后向 Leader 发送 ACK，当 Leader 收到大多数（n/2+1）Follower 的 ACK 信息后将该日志设置为已提交并追加到本地磁盘中，通知客户端并在下个 heartbeat 中 Leader 将通知所有的 Follower 将该日志存储在自己的本地磁盘中
```

15. k8s新增节点准入原理
```
Kubernetes提供一个 certificates.k8s.io API，可让您配置 由您控制的证书颁发机构（CA）签名的TLS证书。 您的工作负载可以使用这些CA和证书来建立信任。

Kubernetes 管理员（具有适当权限）可以使用 kubectl certificate approve 和 kubectl certificate deny 命令手动批准（或拒绝）证书签名请求。但是，如果您打算大量使用此 API，则可以考虑编写自动化的证书控制器。

无论上述机器或人使用 kubectl，批准者的作用是验证 CSR 满足如下两个要求：

CSR 的主体控制用于签署 CSR 的私钥。这解决了伪装成授权主体的第三方的威胁。在上述示例中，此步骤将验证该 pod 控制了用于生成 CSR 的私钥。
CSR 的主体被授权在请求的上下文中执行。这解决了我们加入群集的我们不期望的主体的威胁。在上述示例中，此步骤将是验证该 pod 是否被允许加入到所请求的服务中。
当且仅当满足这两个要求时，审批者应该批准 CSR，否则拒绝 CSR。
```

16. k8s ReplactionController 原理

```
ReplicationController 确保在任何时候都有特定数量的 pod 副本处于运行状态。
当pod 数量过多时，ReplicationController 会终止多余的 pod。
当 pod 数量太少时，ReplicationController 将会启动新的 pod。
与手动创建的 pod 不同，由 ReplicationController 创建的 pod 在失败、被删除或被终止时会被自动替换。 
例如，在中断性维护（如内核升级）之后，您的 pod 会在节点上重新创建。 因此，即使您的应用程序只需要一个 pod，您也应该使用 ReplicationController 创建 Pod。 ReplicationController 类似于进程管理器，但是 ReplicationController 不是监控单个节点上的单个进程，而是监控跨多个节点的多个 pod。

在讨论中，ReplicationController 通常缩写为 "rc"，并作为 kubectl 命令的快捷方式。

一个简单的示例是创建一个 ReplicationController 对象来可靠地无限期地运行 Pod 的一个实例。 更复杂的用例是运行一个多副本服务（如 web 服务器）的若干相同副本。
```

17. spring cloud 在k8s 中通信

```
1.去掉原有的 Eureka，改用 Spring Cloud Kubernetes 项目下的 Discovery。Spring Cloud 官方推出的项目 Spring Cloud Kubernetes 提供了通用的接口来调用Kubernetes服务，让 Spring Cloud 和 Spring Boot 程序能够在 Kubernetes 环境中更好运行。在 Kubernetes 环境中，ETCD 已经拥有了服务发现所必要的信息，没有必要再使用 Eureka，通过 Discovery 就能够获取 Kubernetes ETCD 中注册的服务列表进行服务发现。

2.去掉 Feign 负载均衡，改用 Spring Cloud Kubernetes Ribbon。Ribbon 负载均衡模式有 Service / Pod 两种，在 Service 模式下，可以使用 Kubernetes 原生负载均衡，并通过 Istio 实现服务治理。

3. 网关边缘化。网关作为原来的入口，全部去除需要对原有代码进行大规模的改造，我们把原有的网关作为微服务部署在 Kubernetes 内，并利用 Istio 来管理流量入口。同时，我们还去掉了熔断器和智能路由，整体基于 Istio 实现服务治理。

4. 分布式配置 Config 统一为 Apollo。Apollo 能够集中管理应用在不同环境、不同集群的配置，修改后实时推送到应用端，并且具备规范的权限、流程治理等特性。

5. 增加 Prometheus 监控。特别是对 JVM 一些参数和一些定义指标的监控，并基于监控指标实现了 HPA 弹性伸缩。
```

18. k8s 多集群管理方案

```
Rancher
https://www.rancher.cn/products/rancher/
```

