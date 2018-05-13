## 学习计划
### zookeeper是什么
ZooKeeper是一个分布式的，开放源码的分布式应用程序协调服务，是Google的Chubby一个开源的实现，是Hadoop和Hbase的重要组件。它是一个为分布式应用提供一致性服务的软件，提供的功能包括：配置维护、域名服务、分布式同步、组服务等。
### zookeeper提供了什么
简单的说，zookeeper=文件系统+通知机制。
**1. zookeeper维护了一个类似文件系统的数据结构**
```
/
|---/NameService
|        |---/Server1
|        |---/Server2
|
|---/Configuration
|
|---/GroupMembers
|        |---Member1
|        |---Member2
|
|---/Apps
      |---/App1
      |---/App2
      |---/App3
            |---SubApp1
            |---SubApp2
```
每个子目录项如 ```NameService``` 都被称作为 znode，和文件系统一样，我们能够自由的增加、删除znode，在一个znode下增加、删除子znode，唯一的不同在于znode是可以存储数据的。<br/>
有四种类型的znode：
1. PERSITENT-持久化目录节点<br/>
  客户端与zookeeper断开连接后，改节点依然存在
2. PERSITENT_SEQUENTIAL-持久化顺序编号目录节点<br/>
  客户端与zookeeper断开连接后，该节点依然存在，只是zookeeper给该节点进行顺序编号
3. EPHEMERAL-临时目录节点<br/>
  客户端与zookeeper断开连接后，改节点被删除
4. EPHEMERAIL_SEQUENTIAL-临时顺序目录编号节点<br>
  客户端与zookeeper断开连接后，该节点被删除，只是Zookeeper给该节点名称进行顺序编号<br/>

**2. 通知机制**
客户端注册监听它关心的目录节点，当目录节点发生变化（数据改变、被删除、子目录节点增加删除）时，zookeeper会通知客户端。 
### zookeeper能做什么
**1. 命名服务**
zookeeper的命名服务有两个应用方向，一个是提供类似JNDI的功能，利用zookeepeer的树型分层结构，可以把系统中各种服务的名称、地址以及目录信息存放在zookeeper，需要的时候去zookeeper中读取。

另一个，是利用zookeeper顺序节点的特性，制作分布式的ID生成器，写过数据库应用的朋友都知道，我们在往数据库表中插入记录时，通常需要为该记录创建唯一的ID，在单机环境中我们可以利用数据库的主键自增功能。但在分布式环境则无法使用，有一种方式可以使用UUID，但是它的缺陷是没有规律，很难理解。利用zookeeper顺序节点的特性，我们可以生成有顺序的，容易理解的，同时支持分布式环境的序列号。
**2. 配置管理**
程序总是需要配置的，如果程序分散部署在多台机器上，要逐个改变配置就变得困难。把这些配置全部放到zookeeper上去，保存在Zookeeper的某个目录节点中，然后所有相关应用程序对这个目录节点进行监听，一旦配置信息发生变化，每个应用程序就会收到 Zookeeper 的通知，然后从Zookeeper获取新的配置信息应用到系统中就好。

```
graph TD
A[/configuration]---B[client1]
A---C[client2]
A---D[client3]
```
**3. 集群管理**
Zookeeper 能够很容易的实现集群管理的功能，如有多台 Server 组成一个服务集群，那么必须要一个“总管”知道当前集群中每台机器的服务状态，一旦有机器不能提供服务，集群中其它集群必须知道，从而做出调整重新分配服务策略。同样当增加集群的服务能力时，就会增加一台或多台 Server，同样也必须让“总管”知道。

Zookeeper 不仅能够帮你维护当前的集群中机器的服务状态，而且能够帮你选出一个“总管”，让这个总管来管理集群，这就是 Zookeeper 的另一个功能 Leader Election。

它们的实现方式都是在 Zookeeper 上创建一个 EPHEMERAL 类型的目录节点，然后每个 Server 在它们创建目录节点的父目录节点上调用 getChildren(String path, boolean watch) 方法并设置 watch 为 true，由于是 EPHEMERAL 目录节点，当创建它的 Server 死去，这个目录节点也随之被删除，所以 Children 将会变化，这时 getChildren上的 Watch 将会被调用，所以其它 Server 就知道已经有某台 Server 死去了。新增 Server 也是同样的原理。

Zookeeper 如何实现 Leader Election，也就是选出一个 Master Server。和前面的一样每台 Server 创建一个 EPHEMERAL 目录节点，不同的是它还是一个 SEQUENTIAL 目录节点，所以它是个 EPHEMERAL_SEQUENTIAL 目录节点。之所以它是 EPHEMERAL_SEQUENTIAL 目录节点，是因为我们可以给每台 Server 编号，我们可以选择当前是最小编号的 Server 为 Master，假如这个最小编号的 Server 死去，由于是 EPHEMERAL 节点，死去的 Server 对应的节点也被删除，所以当前的节点列表中又出现一个最小编号的节点，我们就选择这个节点为当前 Master。这样就实现了动态选择 Master，避免了传统意义上单 Master 容易出现单点故障的问题。
![集群管理](/Users/wangbo/typora/study/images/image003.gif)
关键代码:

```java
void findLeader() throws InterruptedException { 
       byte[] leader = null; 
       try { 
           leader = zk.getData(root + "/leader", true, null); 
       } catch (Exception e) { 
           logger.error(e); 
       } 
       if (leader != null) { 
           following(); 
       } else { 
           String newLeader = null; 
           try { 
               byte[] localhost = InetAddress.getLocalHost().getAddress(); 
               newLeader = zk.create(root + "/leader", localhost, 
               ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL); 
           } catch (Exception e) { 
               logger.error(e); 
           } 
           if (newLeader != null) { 
               leading(); 
           } else { 
               mutex.wait(); 
           } 
       } 
   }
```
**4.共享锁**
共享锁在同一个进程中很容易实现，但是在跨进程或者在不同Server之间就不好实现了。Zookeeper 却很容易实现这个功能，实现方式也是需要获得锁的 Server 创建一个 ```EPHEMERAL_SEQUENTIAL```目录节点，然后调用getChildren方法获取当前的目录节点列表中最小的目录节点是不是就是自己创建的目录节点，如果正是自己创建的，那么它就获得了这个锁，如果不是那么它就调用```exists(String path, boolean watch)```方法并监控 Zookeeper上目录节点列表的变化，一直到自己创建的节点是列表中最小编号的目录节点，从而获得锁，释放锁很简单，只要删除前面它自己所创建的目录节点就行了。
Zookeeper 实现 Locks 的流程图：
```
graph TD
A[要获得锁的client<br/>L=create< /Locks/Lock_i, EPHEMERAL_SEQUENTAIL> ]-->B[getChildren< /Locks/ false><br>取得最小值i]
B-->C{当前的最小值i<br>是否等于自己创建的值L}
C-->|否|D[监控比自己创建L小的值j<br> exists< /Locks/Lock_j,true >]
D-->E{/Locks/Lock_j是否存在}
E-->|是|F[等待exists< /Locks/Lock_j,true >中watch的通知]
F-->E
E-->|否|B
C-->|是|G((获得锁))
```
同步锁关键代码如下：
```java
void getLock() throws KeeperException, InterruptedException{ 
       List<String> list = zk.getChildren(root, false); 
       String[] nodes = list.toArray(new String[list.size()]); 
       Arrays.sort(nodes); 
       if(myZnode.equals(root+"/"+nodes[0])){ 
           doAction(); 
       } 
       else{ 
           waitForLock(nodes[0]); 
       } 
   } 
   void waitForLock(String lower) throws InterruptedException, KeeperException {
       Stat stat = zk.exists(root + "/" + lower,true); 
       if(stat != null){ 
           mutex.wait(); 
       } 
       else{ 
           getLock(); 
       } 
   }
```
**5.队列管理**
Zookeeper 可以处理两种类型的队列：

- 1. 当一个队列的成员都聚齐时，这个队列才可用，否则一直等待所有成员到达，这种是同步队列。
- 2. 队列按照 FIFO 方式进行入队和出队操作，例如实现生产者和消费者模型。

**同步队列用 Zookeeper 实现的实现思路如下：**

创建一个父目录/synchronizing，每个成员都监控标志（SetWatch）位目录/synchronizing/start是否存在，然后每个成员都加入这个队列，加入队列的方式就是创建 /synchronizing/member_i 的临时目录节点，然后每个成员获取 /synchronizing 目录的所有目录节点，也就是 member_i。判断 i 的值是否已经是成员的个数，如果小于成员个数等待 /synchronizing/start 的出现，如果已经相等就创建 /synchronizing/start。

加入队列:
```java
void addQueue() throws KeeperException, InterruptedException{ 
       zk.exists(root + "/start",true); 
       zk.create(root + "/" + name, new byte[0], Ids.OPEN_ACL_UNSAFE, 
       CreateMode.EPHEMERAL_SEQUENTIAL); 
       synchronized (mutex) { 
           List<String> list = zk.getChildren(root, false); 
           if (list.size() < size) { 
               mutex.wait(); 
           } else { 
               zk.create(root + "/start", new byte[0], Ids.OPEN_ACL_UNSAFE,
                CreateMode.PERSISTENT); 
           } 
       } 
}
```
当队列没满是进入 wait()，然后会一直等待 Watch 的通知，Watch 的代码如下：
```java
public void process(WatchedEvent event) { 
       if(event.getPath().equals(root + "/start") &&
        event.getType() == Event.EventType.NodeCreated){ 
           super.process(event); 
           doAction(); 
       } 
   }
```
FIFO 队列用 Zookeeper 实现思路如下：

实现的思路也非常简单，就是在特定的目录下创建SEQUENTIAL类型的子目录/queue_i，这样就能保证所有成员加入队列时都是有编号的，出队列时通过getChildren()方法可以返回当前所有的队列中的元素，然后消费其中最小的一个，这样就能保证 FIFO。

生产者代码：
```java
boolean produce(int i) throws KeeperException, InterruptedException{ 
       ByteBuffer b = ByteBuffer.allocate(4); 
       byte[] value; 
       b.putInt(i); 
       value = b.array(); 
       zk.create(root + "/element", value, ZooDefs.Ids.OPEN_ACL_UNSAFE, 
                   CreateMode.PERSISTENT_SEQUENTIAL); 
       return true; 
   }
```
消费者代码:
```java
int consume() throws KeeperException, InterruptedException{ 
       int retvalue = -1; 
       Stat stat = null; 
       while (true) { 
           synchronized (mutex) { 
               List<String> list = zk.getChildren(root, true); 
               if (list.size() == 0) { 
                   mutex.wait(); 
               } else { 
                   Integer min = new Integer(list.get(0).substring(7)); 
                   for(String s : list){ 
                       Integer tempValue = new Integer(s.substring(7)); 
                       if(tempValue < min) min = tempValue; 
                   } 
                   byte[] b = zk.getData(root + "/element" + min,false, stat); 
                   zk.delete(root + "/element" + min, 0); 
                   ByteBuffer buffer = ByteBuffer.wrap(b); 
                   retvalue = buffer.getInt(); 
                   return retvalue; 
               } 
           } 
       } 
}
```
### [zookeeper原理](https://blog.csdn.net/u010039929/article/details/70171672)


