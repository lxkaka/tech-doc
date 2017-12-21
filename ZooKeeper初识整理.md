
### 整体把握
* zk 是针对分布式系统的协调服务（本身就是分布式应用程序），优点是可靠，可扩展，高性能。
* 遵循 C/S 模型。(这里C就是我们使用zk服务的机器，S自然就是提供zk服务)。zk可以提供单机服务，也可组成集群提供服务，还支持伪集群方式(一台物理机运行多个zookeeper实例）。客户端连接到一个单独的服务。客户端保持了一个TCP连接，通过这个TCP连接发送请求、获取响应、获取watch事件、和发送心跳。如果这个连接断了，会自动连接到其他不同的服务器。  
下图我简单画了一下zk的工作模型
![zk工作模型](http://7xorjs.com1.z0.glb.clouddn.com/zk_cs_model.png)
* 	zk的数据模型类似文件系统，由 znode 组成目录树的形式，每个节点下可以有子节点。
	节点可以是以下四种类型：  
	* `PERSISTENT`：持久化目录节点，这个目录节点存储的数据不会丢失；
	* `PERSISTENT_SEQUENTIAL`：顺序自动编号的目录节点，这种目录节点会根据当前已近存在的节点数自动加 1，然后返回给客户端已经成功创建的目录节点名；
	* `EPHEMERAL`：临时目录节点，一旦创建这个节点的客户端与服务器端口也就是 session 超时，这种节点会被自动删除；
	* `EPHEMERAL_SEQUENTIAL`：临时自动编号节点。
监控节点变化时，可以监控一个节点的变化，也可以监控一个节点所有子节点的变化。zk一些很重要的应用都是依赖这些节点的特性。  
下图我简单画了一下zk的node结构
![zk node结构](http://7xorjs.com1.z0.glb.clouddn.com/zk_node.png)

### ZK的主要应用场景
* 配置管理（Configuration Management）  
	配置的管理在分布式应用环境中很常见，例如同一个应用系统需要多台 Server 运行，但是它们运行的应用系统的某些配置项是相同的，如果要修改这些相同的配置项，那么就必须同时修改每台运行这个应用系统的  Server，这样非常麻烦而且容易出错。  
像这样的配置信息完全可以交给 Zookeeper 来管理，将配置信息保存在 Zookeeper 的某个目录节点中，然后将所有需要修改的应用机器监控配置信息的状态，一旦配置信息发生变化，每台应用机器就会收到 Zookeeper 的通知，然后从 Zookeeper 获取新的配置信息应用到系统中。  
下图是配置管理的结构图（引用）
![zk配置管理](http://7xorjs.com1.z0.glb.clouddn.com/zk_config.png)

* 集群管理（Group Membership)     
	Zookeeper 能够很容易的实现集群管理的功能，如有多台 Server 组成一个服务集群，那么必须要一个“总管”知道当前集群中每台机器的服务状态，一旦有机器不能提供服务，集群中其它集群必须知道，从而做出调整重新分配服务策略。同样当增加集群的服务能力时，就会增加一台或多台 Server，同样也必须让“总管”知道。  
	Zookeeper 不仅能够帮你维护当前的集群中机器的服务状态，而且能够帮你选出一个“总管”，让这个总管来管理集群，这就是 Zookeeper 的另一个功能 Leader Election。  
	它们的实现方式都是在 Zookeeper 上创建一个 EPHEMERAL 类型的目录节点，然后每个 Server 在它们创建目录节点的父目录节点上调用`getChildren(String path, boolean watch)` 方法并设置 watch 为 true，由于是 `EPHEMERAL` 目录节点，当创建它的 Server 死去，这个目录节点也随之被删除，所以 Children 将会变化，这时 `getChildren`上的 Watch 将会被调用，所以其它 Server 就知道已经有某台 Server 死去了。新增 Server 也是同样的原理。  
	Zookeeper 如何实现 Leader Election，也就是选出一个 Master Server。和前面的一样每台 Server 创建一个 `EPHEMERAL` 目录节点，不同的是它还是一个 `SEQUENTIAL` 目录节点，所以它是个 `EPHEMERAL_SEQUENTIAL` 目录节点。之所以它是 `EPHEMERAL_SEQUENTIAL` 目录节点，是因为我们可以给每台 Server 编号，我们可以选择当前是最小编号的 Server 为 Master，假如这个最小编号的 Server 死去，由于是 `EPHEMERA`L 节点，死去的 Server 对应的节点也被删除，所以当前的节点列表中又出现一个最小编号的节点，我们就选择这个节点为当前 Master。这样就实现了动态选择 Master，避免了传统意义上单 Master 容易出现单点故障的问题。  
下图是集群管理的原理示意
![zk集群管理](http://7xorjs.com1.z0.glb.clouddn.com/zk_group.png)
	
* 共享锁（Locks）  
	共享锁在同一个进程中很容易实现，但是在跨进程或者在不同 Server 之间就不好实现了。Zookeeper 却很容易实现这个功能，实现方式也是需要获得锁的 Server 创建一个 `EPHEMERAL_SEQUENTIAL` 目录节点，然后调用 `getChildren`方法获取当前的目录节点列表中最小的目录节点是不是就是自己创建的目录节点，如果正是自己创建的，那么它就获得了这个锁，如果不是那么它就调用 `exists(String path, boolean watch)` 方法并监控 Zookeeper 上目录节点列表的变化，一直到自己创建的节点是列表中最小编号的目录节点，从而获得锁，释放锁很简单，只要删除前面它自己所创建的目录节点就行了。

* 	队列管理  
	Zookeeper 可以处理两种类型的队列：  
		1. 当一个队列的成员都聚齐时，这个队列才可用，否则一直等待所有成员到达，这种是同步队列。  
		2. 队列按照 FIFO 方式进行入队和出队操作，例如实现生产者和消费者模型。  
	创建一个父目录 `/synchronizing`，每个成员都监控标志（Set Watch）位目录 /`synchronizing/start` 是否存在，然后每个成员都加入这个队列，加入队列的方式就是创建 /`synchronizing/member_i` 的临时目录节点，然后每个成员获取 `/synchronizing` 目录的所有目录节点，也就是 member_i。判断 i 的值是否已经是成员的个数，如果小于成员个数等待 `/synchronizing/start` 的出现，如果已经相等就创建 `/synchronizing/start`。  
	FIFO 队列用 Zookeeper 实现思路如下：  
	在特定的目录下创建 SEQUENTIAL 类型的子目录 /queue_i，这样就能保证所有成员加入队列时都是有编号的，出队列时通过 getChildren( ) 方法可以返回当前所有的队列中的元素，然后消费其中最小的一个，这样就能保证 FIFO。

### ZK简单实践
* mac下 `brew install zookeeper`  
在集合体中，可以包含一个节点，但它不是一个高可用和可靠的系统。如果在集合体中有两个节点，那么这两个节点都必须已经启动并让服务正常运行，因为两个节点中的一个并不是严格意义上的多数。如果在集合体中有三个节点，即使其中一个停机了，您仍然可以获得正常运行的服务（三个中的两个是严格意义上的多数）。出于这个原因，ZooKeeper 的集合体中通常包含奇数数量的节点，因为就容错而言，与三个节点相比，四个节点并不占优势，因为只要有两个节点停机，ZooKeeper 服务就会停止。在有五个节点的集群上，需要三个节点停机才会导致 ZooKeeper 服务停止运作。
* 搭建3个zk实例（伪集群配置）  
	* 在 usr/l/ocal/var/run/zookeeper/ 下新建文件夹 zk1, zk2, zk3
	* 在 /usr/local/etc/zookeeper 下 新建 zk1.cfg,  zk2.cfg,  zk3.cfg 
	* 修改项	
	
		```
		dataDir=/usr/local/var/run/zookeeper/zk1
		# the port at which the clients will connect
		clientPort=2181
		server.1=localhost:2888:3888
		server.2=localhost:2889:3889
		server.3=localhost:2890:3890
		```
	* 在文件夹 /usr/local/var/run/zookeeper/zk1 下 `echo "1" > /usr/local/var/run/zookeeper/zk1/myid`  
	zk2 下 `echo "2" > myid`    `zk3 下 echo "3" > myid`; <font color='red'>myid</font>里面的值与 配置 文件里的 server. <font color='red'>编号</font>一致
	* 依次启动：  
		`zkServer start /usr/local/etc/zk1.cfg`	 	`zkServer start /usr/local/etc/zk2.cfg`
		`zkServer start /usr/local/etc/zk3.cfg`
	* 检查状态：
		`zkServer status /usr/local/etc/zk1.cfg`  
		输出 ：  
		
		```
		ZooKeeper JMX enabled by default
		Using config: /usr/local/etc/zookeeper/zk1.cfg
		Mode: follower
		```
	* 创建一个znode   
		创建一个新znode和相关联的数据:  `create /test_data  test1234`
		用 get 获取数据： `get /test_data`；  
		用 set 修改： `set/test_data  abcd1234`  
		连接到其他 zk 服务器，可获取到同样的数据 :  `get /test_data 1`
		这里在最后提供了一个可选参数 **1**。此参数为 /tes_data 上的数据设置了一个一次性的触发		器（名称为 watch）。如果另一个客户端在 /test_data 上修改数据，该客户端将会获得一		个异步通知。请注意，该通知只发送一次，除非 watch 被重新设置，否则不会因数据发生改变		而再次发送通知。  
	如图所示：
	![command line](http://7xorjs.com1.z0.glb.clouddn.com/zk_watch.png)
* 利用 python 实现的 zookeeper 客户端 kazoo 与 zk server 交互

	* 创建临时自动编号节点  
		
		```python
		from kazoo.client import KazooClient

		hosts = ['127.0.0.1:12181', '127.0.0.1:12182', '127.0.0.1:12183']
		zk = KazooClient(host=hosts[0])
		zk.start()
		print(zk.state)
		# 创建一个目录节点
		zk.create('/seq_test')
		# 创建一个临时自动编号子目录节点
		zk.create('/seq_test/test_i', ephemeral=True, sequence=True)
		print(zk.get('/seq_test'))
		# 获取子节点信息
		child = zk.get_children('/seq_test')
		```
	* 定义一个 watch event,  watch 一个节点的变化

		```python
		def watch_event（child):
		    print('action triggered：', child)
		
		zk.create('/election')
		child = zk.get_children('/election', watch=watch_event)
		zk.create('/election/child')
		zk.delete('/election/child')
		```
	结果示意图
	![结果示意](http://7xorjs.com1.z0.glb.clouddn.com/zk_result.png)
