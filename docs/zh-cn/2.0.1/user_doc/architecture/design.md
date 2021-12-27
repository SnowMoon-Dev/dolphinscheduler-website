## 系统架构设计
本章节介绍Apache DolphinScheduler调度系统架构


### 1.系统架构

#### 1.1 系统架构图
<p align="center">
  <img src="/img/architecture-1.3.0.jpg" alt="系统架构图"  width="70%" />
  <p align="center">
        <em>系统架构图</em>
  </p>
</p>

#### 1.2 启动流程活动图
<p align="center">
  <img src="/img/master-process-2.0-zh_cn.png" alt="Start process activity diagram"  width="70%" />
  <p align="center">
        <em>启动流程活动图</em>
  </p>
</p>

#### 1.3 架构说明

* **MasterServer** 

    MasterServer采用分布式无中心设计理念，MasterServer主要负责 DAG 任务切分、任务提交监控，并同时监听其它MasterServer和WorkerServer的健康状态。
    MasterServer服务启动时向Zookeeper注册临时节点，通过监听Zookeeper临时节点变化来进行容错处理。
    MasterServer基于netty提供监听服务。

    ##### 该服务内主要包含:

    - **Distributed Quartz**分布式调度组件，主要负责定时任务的启停操作，当quartz调起任务后，Master内部会有线程池具体负责处理任务的后续操作

    - **MasterSchedulerService**是一个扫描线程，定时扫描数据库中的 **command** 表，生成工作流实例，根据不同的**命令类型**进行不同的业务操作

    - **WorkflowExecuteThread**主要是负责DAG任务切分、任务提交、各种不同命令类型的逻辑处理，处理任务状态和工作流状态事件

    - **EventExecuteService**处理master负责的工作流实例所有的状态变化事件，使用线程池处理工作流的状态事件
    
    - **StateWheelExecuteThread**处理依赖任务和超时任务的定时状态更新

* **WorkerServer** 

     WorkerServer也采用分布式无中心设计理念，支持自定义任务插件，主要负责任务的执行和提供日志服务。
     WorkerServer服务启动时向Zookeeper注册临时节点，并维持心跳。
     
     ##### 该服务包含：
     
     - **WorkerManagerThread**主要通过netty领取master发送过来的任务，并根据不同任务类型调用**TaskExecuteThread**对应执行器。
     
     - **RetryReportTaskStatusThread**主要通过netty向master汇报任务状态，如果汇报失败，会一直重试汇报

     - **LoggerServer**是一个日志服务，提供日志分片查看、刷新和下载等功能
     
* **Registry** 

    注册中心，使用插件化实现，默认支持Zookeeper, 系统中的MasterServer和WorkerServer节点通过注册中心来进行集群管理和容错。另外系统还基于注册中心进行事件监听和分布式锁。
    
* **Alert** 

    提供告警相关功能，仅支持单机服务。支持自定义告警插件。

* **API** 

    API接口层，主要负责处理前端UI层的请求。该服务统一提供RESTful api向外部提供请求服务。
    接口包括工作流的创建、定义、查询、修改、发布、下线、手工启动、停止、暂停、恢复、从该节点开始执行等等。

* **UI** 

    系统的前端页面，提供系统的各种可视化操作界面，详见<a href="/zh-cn/docs/2.0.1/user_doc/system-manual.html" target="_self">系统使用手册</a>部分。

#### 1.4 架构设计思想

##### 一、去中心化vs中心化 

###### 中心化思想

中心化的设计理念比较简单，分布式集群中的节点按照角色分工，大体上分为两种角色：
<p align="center">
   <img src="https://analysys.github.io/easyscheduler_docs_cn/images/master_slave.png" alt="master-slave角色"  width="50%" />
 </p>

- Master的角色主要负责任务分发并监督Slave的健康状态，可以动态的将任务均衡到Slave上，以致Slave节点不至于“忙死”或”闲死”的状态。
- Worker的角色主要负责任务的执行工作并维护和Master的心跳，以便Master可以分配任务给Slave。



中心化思想设计存在的问题：

- 一旦Master出现了问题，则群龙无首，整个集群就会崩溃。为了解决这个问题，大多数Master/Slave架构模式都采用了主备Master的设计方案，可以是热备或者冷备，也可以是自动切换或手动切换，而且越来越多的新系统都开始具备自动选举切换Master的能力,以提升系统的可用性。
- 另外一个问题是如果Scheduler在Master上，虽然可以支持一个DAG中不同的任务运行在不同的机器上，但是会产生Master的过负载。如果Scheduler在Slave上，则一个DAG中所有的任务都只能在某一台机器上进行作业提交，则并行任务比较多的时候，Slave的压力可能会比较大。



###### 去中心化
 <p align="center">
   <img src="https://analysys.github.io/easyscheduler_docs_cn/images/decentralization.png" alt="去中心化"  width="50%" />
 </p>

- 在去中心化设计里，通常没有Master/Slave的概念，所有的角色都是一样的，地位是平等的，全球互联网就是一个典型的去中心化的分布式系统，联网的任意节点设备down机，都只会影响很小范围的功能。
- 去中心化设计的核心设计在于整个分布式系统中不存在一个区别于其他节点的”管理者”，因此不存在单点故障问题。但由于不存在” 管理者”节点所以每个节点都需要跟其他节点通信才得到必须要的机器信息，而分布式系统通信的不可靠性，则大大增加了上述功能的实现难度。
- 实际上，真正去中心化的分布式系统并不多见。反而动态中心化分布式系统正在不断涌出。在这种架构下，集群中的管理者是被动态选择出来的，而不是预置的，并且集群在发生故障的时候，集群的节点会自发的举行"会议"来选举新的"管理者"去主持工作。最典型的案例就是ZooKeeper及Go语言实现的Etcd。


- DolphinScheduler的去中心化是Master/Worker注册到Zookeeper中，实现Master集群和Worker集群无中心，使用分片机制，公平分配工作流在master上执行，并通过不同的发送策略将任务发送给worker执行具体的任务

#####  二、Master执行流程

1. DolphinScheduler使用分片算法将command取模，根据master的排序id分配，master将拿到的command转换成工作流实例，使用线程池处理工作流实例


2. DolphinScheduler对工作流的处理流程:

  - 通过UI或者API调用，启动工作流，持久化一条command到数据库中
  - Master通过分片算法，扫描Command表，生成工作流实例ProcessInstance，同时删除Command数据
  - Master使用线程池运行WorkflowExecuteThread，执行工作流实例的流程，包括构建DAG，创建任务实例TaskInstance，将TaskInstance通过netty发送给worker
  - Worker收到任务以后，修改任务状态，并将执行信息返回Master
  - Master收到任务信息，持久化到数据库，并且将状态变化事件存入EventExecuteService事件队列
  - EventExecuteService根据事件队列调用WorkflowExecuteThread进行后续任务的提交和工作流状态的修改


##### 三、容错设计
容错分为服务宕机容错和任务重试，服务宕机容错又分为Master容错和Worker容错两种情况

###### 1. 宕机容错

服务容错设计依赖于ZooKeeper的Watcher机制，实现原理如图：

 <p align="center">
   <img src="https://analysys.github.io/easyscheduler_docs_cn/images/fault-tolerant.png" alt="DolphinScheduler容错设计"  width="40%" />
 </p>
其中Master监控其他Master和Worker的目录，如果监听到remove事件，则会根据具体的业务逻辑进行流程实例容错或者任务实例容错。



- Master容错流程图：

 <p align="center">
   <img src="https://analysys.github.io/easyscheduler_docs_cn/images/fault-tolerant_master.png" alt="Master容错流程图"  width="40%" />
 </p>
ZooKeeper Master容错完成之后则重新由DolphinScheduler中Scheduler线程调度，遍历 DAG 找到”正在运行”和“提交成功”的任务，对”正在运行”的任务监控其任务实例的状态，对”提交成功”的任务需要判断Task Queue中是否已经存在，如果存在则同样监控任务实例的状态，如果不存在则重新提交任务实例。



- Worker容错流程图：

 <p align="center">
   <img src="https://analysys.github.io/easyscheduler_docs_cn/images/fault-tolerant_worker.png" alt="Worker容错流程图"  width="40%" />
 </p>

Master Scheduler线程一旦发现任务实例为” 需要容错”状态，则接管任务并进行重新提交。

 注意：由于” 网络抖动”可能会使得节点短时间内失去和ZooKeeper的心跳，从而发生节点的remove事件。对于这种情况，我们使用最简单的方式，那就是节点一旦和ZooKeeper发生超时连接，则直接将Master或Worker服务停掉。

###### 2.任务失败重试

这里首先要区分任务失败重试、流程失败恢复、流程失败重跑的概念：

- 任务失败重试是任务级别的，是调度系统自动进行的，比如一个Shell任务设置重试次数为3次，那么在Shell任务运行失败后会自己再最多尝试运行3次
- 流程失败恢复是流程级别的，是手动进行的，恢复是从只能**从失败的节点开始执行**或**从当前节点开始执行**
- 流程失败重跑也是流程级别的，是手动进行的，重跑是从开始节点进行



接下来说正题，我们将工作流中的任务节点分了两种类型。

- 一种是业务节点，这种节点都对应一个实际的脚本或者处理语句，比如Shell节点，MR节点、Spark节点、依赖节点等。

- 还有一种是逻辑节点，这种节点不做实际的脚本或语句处理，只是整个流程流转的逻辑处理，比如子流程节等。

所有任务都可以配置失败重试的次数，当该任务节点失败，会自动重试，直到成功或者超过配置的重试次数。

如果工作流中有任务失败达到最大重试次数，工作流就会失败停止，失败的工作流可以手动进行重跑操作或者流程恢复操作



##### 四、任务优先级设计
在早期调度设计中，如果没有优先级设计，采用公平调度设计的话，会遇到先行提交的任务可能会和后继提交的任务同时完成的情况，而不能做到设置流程或者任务的优先级，因此我们对此进行了重新设计，目前我们设计如下：

-  按照**不同流程实例优先级**优先于**同一个流程实例优先级**优先于**同一流程内任务优先级**优先于**同一流程内任务**提交顺序依次从高到低进行任务处理。
    - 具体实现是根据任务实例的json解析优先级，然后把**流程实例优先级_流程实例id_任务优先级_任务id**信息保存在ZooKeeper任务队列中，当从任务队列获取的时候，通过字符串比较即可得出最需要优先执行的任务

        - 其中流程定义的优先级是考虑到有些流程需要先于其他流程进行处理，这个可以在流程启动或者定时启动时配置，共有5级，依次为HIGHEST、HIGH、MEDIUM、LOW、LOWEST。如下图
            <p align="center">
               <img src="https://analysys.github.io/easyscheduler_docs_cn/images/process_priority.png" alt="流程优先级配置"  width="40%" />
             </p>

        - 任务的优先级也分为5级，依次为HIGHEST、HIGH、MEDIUM、LOW、LOWEST。如下图
            <p align="center">
               <img src="https://analysys.github.io/easyscheduler_docs_cn/images/task_priority.png" alt="任务优先级配置"  width="35%" />
             </p>


##### 五、Logback和netty实现日志访问

-  由于Web(UI)和Worker不一定在同一台机器上，所以查看日志不能像查询本地文件那样。有两种方案：
  -  将日志放到ES搜索引擎上
  -  通过netty通信获取远程日志信息

-  介于考虑到尽可能的DolphinScheduler的轻量级性，所以选择了gRPC实现远程访问日志信息。

 <p align="center">
   <img src="https://analysys.github.io/easyscheduler_docs_cn/images/grpc.png" alt="grpc远程访问"  width="50%" />
 </p>


- 我们使用自定义Logback的FileAppender和Filter功能，实现每个任务实例生成一个日志文件。
- FileAppender主要实现如下：

 ```java
 /**
  * task log appender
  */
 public class TaskLogAppender extends FileAppender<ILoggingEvent> {
 
     ...

    @Override
    protected void append(ILoggingEvent event) {

        if (currentlyActiveFile == null){
            currentlyActiveFile = getFile();
        }
        String activeFile = currentlyActiveFile;
        // thread name： taskThreadName-processDefineId_processInstanceId_taskInstanceId
        String threadName = event.getThreadName();
        String[] threadNameArr = threadName.split("-");
        // logId = processDefineId_processInstanceId_taskInstanceId
        String logId = threadNameArr[1];
        ...
        super.subAppend(event);
    }
}
 ```


以/流程定义id/流程实例id/任务实例id.log的形式生成日志

- 过滤匹配以TaskLogInfo开始的线程名称：

- TaskLogFilter实现如下：

 ```java
 /**
 *  task log filter
 */
public class TaskLogFilter extends Filter<ILoggingEvent> {

    @Override
    public FilterReply decide(ILoggingEvent event) {
        if (event.getThreadName().startsWith("TaskLogInfo-")){
            return FilterReply.ACCEPT;
        }
        return FilterReply.DENY;
    }
}
 ```

