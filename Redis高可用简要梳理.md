Redis因为其高性能和易用性在我们后端的服务中发挥了巨大的作用，并且很多重要功能的实现都会依赖redis。除了常用的缓存，还有队列，发布订阅等重要用处。所以redis的服务高可用就显得尤为关键。这里从redis的高可用入手，梳理了一下目前主流的方案。

## Redis-Sentinel
Redis-Sentinel是Redis官方推荐的高可用性(HA)解决方案，当用Redis做Master-slave的高可用方案时，假如master宕机了，Redis本身(包括它的很多客户端)都没有实现自动进行主备切换，而Redis-sentinel本身也是一个独立运行的进程，它能监控多个master-slave集群，发现master宕机后能进行自动切换。
![单机sentinel](http://7xorjs.com1.z0.glb.clouddn.com/redis-se1.png)

它的主要功能有以下几点

* 不时地监控redis是否按照预期良好地运行;

* 如果发现某个redis节点运行出现状况，能够通知另外一个进程(例如它的客户端);

* 能够进行自动切换。当一个master节点不可用时，能够选举出master的多个slave(如果有超过一个slave的话)中的一个来作为新的master,其它的slave节点会将它所追随的master的地址改为被提升为master的slave的新地址。

### Sentinel支持集群
很显然，只使用单个sentinel进程来监控redis集群是不可靠的，当sentinel进程宕掉后(sentinel本身也有单点问题，single-point-of-failure)整个集群系统将无法按照预期的方式运行。所以有必要将sentinel集群，这样有几个好处：

* 即使有一些sentinel进程宕掉了，依然可以进行redis集群的主备切换；

* 如果只有一个sentinel进程，如果这个进程运行出错，或者是网络堵塞，那么将无法实现redis集群的主备切换（单点问题）;

* 如果有多个sentinel，redis的客户端可以随意地连接任意一个sentinel来获得关于redis集群中的信息。

![集群sentinel](http://7xorjs.com1.z0.glb.clouddn.com/redis-se2.png)

redis-sentinel方案提供了单点的高可用解决方案，但是当数据量和业务量极速增长时，单点的reids不可能无限的纵向扩容（增大内存), 这个时候就需要redis有集群的能力来扛。
redis集群的几种实现方式如下：

* 客户端分片  
	优点简单客户端sharding不支持动态增删节点。劣势很大服务端Redis实例群拓扑结构有变化时，每个客户端都需要更新调整。连接不能共享，当应用规模增大时，资源浪费制约优化。一般不采用。
* 基于代理的分片，如codis和Twemproxy
* 路由查询， redis-cluster 


## Twemproxy
Twemproxy也叫nutcraker，是twtter开源的一个redis和memcache代理服务器程序。redis作为一个高效的缓存服务器，非常具有应用价值。但在用户数据量增大时，需要运行多个redis实例，此时将迫切需要一种工具统一管理多个redis实例，避免在每个客户端管理所有连接带来的不方便和不易维护，Twemproxy即为此目标而生。
![tweemproxy](http://7xorjs.com1.z0.glb.clouddn.com/twemproxy01.png)
### 主要工作方式
分布式逻辑和存储引擎分开，逻辑层proxy代理，存储用的是原子redis，当每个层面需要添加删除节点必须重启服务生效（要重新利用散列函数生成KEY分片更新）  
Proxy无状态，redis数据层有状态的，客户端可以请求任一proxy代理上面，再由其转发至正确的redis节点，该KEY分片算法至某个节点都是预先已经算好的，在proxy配置文件保存着，但是如果更新或者删除节点，又要根据一致性hash重新计算分片，并且重启服务。  
一致性hash算法，增减节点需要配置proxy通知新的算法，重启服务

#### 优点：

* 比较轻，开发简单，对应用几乎透明
* 历史悠久，方案成熟  

#### 缺点：
* 代理影响性能
* 无法平滑地扩容/缩容
* 运维比较困难
* proxy单点本身会有性能瓶颈


## Codis
codis由3大组件构成：

* codis-server : 修改过源码的redis, 支持slot，扩容迁移等
* codis-proxy : 支持多线程，go语言实现的内核
* codis Dashboard : 集群管理工具
提供web图形界面管理集群。
集群元数据存在在zookeeper或etcd。(Zookeeper/etcd存放数据路由表和codis-proxy节点的元信息，codis-config发起的命令通过其同步到各个存活的codis-proxy)
提供独立的组件codis-ha负责redis节点主备切换。
基于proxy的codis，客户端对路由表变化无感知。客户端需要从codis dashhoard调用list proxy命令获取所有proxy列表，并根据自身的轮询策略决定访问哪个proxy节点以实现负载均衡。  
整个集群分为1024个哈希槽，分片算法位SlotId = crc32(key) % 1024，增减节点不需要重启服务
![codis](http://7xorjs.com1.z0.glb.clouddn.com/codis.png)

### 主要工作方式
分布式逻辑和存储引擎分开，逻辑层codis-proxy，存储用的是修改过的codis-server,这种好处是proxy层可以自动扩展和收缩，存储层也同样可以，每个层面都可以热插拨  
proxy无状态，codis-server分为组间，每个组存在一个主节点（必须有并且只能有一个）和多个从节点。客户端请求都是和proxy链接，链接哪个proxy都一样，然后由它根据zookeeper路由信息转发至正确节点，直接可以定位到正确节点上

#### 优点：

* 对应用几乎透明
* 性能比 Twemproxy 好
* 有图形化界面，扩容容易，运维方便

#### 缺点：
* 代理影响性能
* 组件过多，需要很多机器资源，部署比较难
* 修改了 Redis 代码，导致和官方无法同步，新特性跟进缓慢

## Redis Cluster
### 主要工作方式
分布式的逻辑和存储引擎不分开，即又负责读写操作，又负责集群交互，升级困难，如果代码有bug，集群无法工作
这个结构为无中心的组织，不好把控集群当前的存活状态，客户端可以向任一节点发送请求，再有其重定向正确的节点上。如果在第一次请求和重定向期间cluster拓扑结构改变，则需要再一次或者多次重定向至正确的节点，但是这方面性能可以忽悠不计  
整个集群分为16384个哈希槽，分片算法位SlotId = crc16(key) % 16384，增减节点不需要重启服务。Redis 集群通过 Gossip 协议同步节点信息，基本思想是节点之间互相交换信息最终所有节点达到一致。BTW. redis sentinel集群判定redis主节点down掉也是采用gossip协议。关于Gossip参考[wiki](https://en.wikipedia.org/wiki/Gossip_protocol)。

![redis-cluster](http://7xorjs.com1.z0.glb.clouddn.com/redis-cluster.png)

#### 优点：

* 组件 all-in-box，部署简单
* 性能最快（没有proxy）
* 自动故障转移、Slot 迁移中数据可用
* 官方原生集群方案，更新与支持有保障

#### 缺点：

* 实践不多
* 客户端开放成本高（客户端不够成熟）
* 多键操作支持有限
* reshard 操作不够自动化（redis-trib.rb ）
