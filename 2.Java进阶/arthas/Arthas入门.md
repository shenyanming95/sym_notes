---
title: Arthas入门
date: 2019-04-22 22:52:50
tags:
  - Arthas
  - JVM
---

### 1. Arthas为何物
在线排查问题，无需重启；动态跟踪Java代码；实时监控JVM状态。

### 2. 使用入门
#### 2.1. dashboard
```shell
$ dashboard
ID     NAME                GROUP        PRIORI STATE %CPU   TIME   INTER DAEMON 
224814 Timer-for-arthas-da system       10     RUNNA 77     0:0    false true   
159    Thread-10           main         5      TIMED 4      3:7    false false  
198    DubboResponseTimeou main         5      TIMED 1      1:7    false true   
1351   RxComputationSchedu main         5      TIMED 1      0:12   false true   
1857   RxComputationSchedu main         5      TIMED 1      0:14   false true   
211    AMQP Connection 172 main         5      RUNNA 0      0:4    false false  
36     Abandoned connectio main         5      TIMED 0      0:22   false true   
288    AnonymousIoService- main         5      WAITI 0      0:0    false true   
408    AnonymousIoService- main         5      WAITI 0      0:0    false true   
409    AnonymousIoService- main         5      WAITI 0      0:0    false true   
Memory           used  total max  usage GC                                      
heap             611M  1183M                                1228                
ps_eden_space    282M  312M  651M       gc.ps_scavenge.time 8073                
                 11M   13M   13M        (ms)                                    
ps_old_gen       318M  857M             gc.ps_marksweep.cou 4                   
nonheap          226M  233M  -1         nt                                      
Runtime                                                                         
os.name             Linux                                                       
os.version          4.4.0-141-generic                                           
java.version        1.8.0_181                                                   
java.home           /usr/lib/jvm/java-8                                         
                    -openjdk-amd64/jre 
```

- ID: Java级别的线程ID，注意这个ID不能跟jstack中的nativeID一一对应
- NAME: 线程名
- GROUP: 线程组名
- PRIORITY: 线程优先级, 1~10之间的数字，越大表示优先级越高
- STATE: 线程的状态
- CPU%: 线程消耗的cpu占比，采样100ms，将所有线程在这100ms内的cpu使用量求和，再算出每个线程的cpu使用占比。
- TIME: 线程运行总时间，数据格式为分：秒
- INTERRUPTED: 线程当前的中断位状态
- DAEMON: 是否是daemon线程
#### 2.2. thread
查看当前线程信息，查看线程的堆栈

这里的cpu统计的是，一段采样间隔内，当前JVM里各个线程所占用的cpu时间占总cpu时间的百分比。其计算方法为： 首先进行一次采样，获得所有线程的cpu的使用时间(调用的是java.lang.management.ThreadMXBean#getThreadCpuTime这个接口)，然后睡眠一段时间，默认100ms，可以通过-i参数指定，然后再采样一次，最后得出这段时间内各个线程消耗的cpu时间情况，最后算出百分比。
注意： 这个统计也会产生一定的开销（JDK这个接口本身开销比较大），因此会看到as的线程占用一定的百分比，为了降低统计自身的开销带来的影响，可以把采样间隔拉长一些，比如5000毫秒。

#### 2.3. watch
让你能方便的观察到指定方法的调用情况。能观察到的范围为：返回值、抛出异常、入参，通过编写 OGNL 表达式进行对应变量的查看。

- watch 命令定义了4个观察事件点，即 -b 方法调用前，-e 方法异常后，-s 方法返回后，-f 方法结束后（包括异常返回和正常返回）
- 4个观察事件点 -b、-e、-s 默认关闭，-f 默认打开，当指定观察点被打开后，在相应事件点会对观察表达式进行求值并输出
- 这里要注意方法入参和方法出参的区别，有可能在中间被修改导致前后不一致，除了 -b 事件点 params 代表方法入参外，其余事件都代表方法出参
- 当使用 -b 时，由于观察事件点是在方法调用前，此时返回值或异常均不存在

#### 2.4. trace
方法内部调用路径，并输出方法路径上的每个节点上耗时

trace 命令能主动搜索 class-pattern／method-pattern 对应的方法调用路径，渲染和统计整个调用链路上的所有性能开销和追踪调用链路

#### 2.5. stack
输出当前方法被调用的调用路径

很多时候我们都知道一个方法被执行，但这个方法被执行的路径非常多，或者你根本就不知道这个方法是从那里被执行了，此时你需要的是 stack 命令。

### 3. 实际案例
#### 3.1. 进入Docker内部的Arthas
1. 进入Docker
```shell
sudo su -
docker ps
docker exec -it inapideploy_notification-service_1 bash
```
2. 进入Arthas
```shell
java -jar /app/arthas-boot.jar --repo-mirror aliyun --use-http



pupu_main.activityredeemproducts,pupu_main.activitylimitstores,pupu_main.activitycategories,pupu_main.activityentities,pupu_main.activitylimitproducts,pupu_main.activitylimitcities
```
#### 3.2. 方法的参数，返回值和异常
1. 参数和返回值
```shellcom
watch com.pupu.order.state.logistics.WaitConfirmLogisticsTaskState nextStep "{params, returnObj}" -x 2

Press Q or Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 322 ms.
ts=2019-04-23 16:46:31; [cost=212.880012ms] result=@ArrayList[
    @Object[][isEmpty=false;size=4],
    @LogisticsTasks[LogisticsTasks(id=9fba1974-8165-4b68-abf6-b84ef273ca52, taskNum=000101556008897002, storeId=6660fb8b-9a97-417c-ab47-1ef9c102e64a, placeId=9b9110be-03d8-4f6a-9274-39639148346c, storeDistance=1422.0, logisticsType=10, sourceType=0, sourceId=9fba1974-8165-4b68-abf6-b84ef273ca52, sourceNum=0101556008897002, buyerId=ea356bd0-4da9-4888-9a4b-cd4e9cd2da81, logisticsStatus=110, deliveryTimeType=0, timeDeliveryPromise=1556010702440, isHasReceipt=false, executerId=0c4f156b-d69a-4bf8-88cf-c164b07893e7, executerName=余秀禹, executerPhone=17605009769, taskStatus=10, totalPrice=3948, remark=新版调度系统：手动抢单, userIdCreate=00000000-0000-0000-0000-000000000000, timeCreate=1556008902440, timeUpdate=1556009190781, userIdUpdate=00000000-0000-0000-0000-000000000000, receiverId=12012559-3dc4-432d-93e6-165130e03d78, isPrint=true, processStatus=10, internalTaskNum=H3164135, timeDeliveryPromiseStart=1556010702440, timeDeliveryPromiseEnd=1556010702440, productWeight=8835, isReady=true, sortNum=0)],
]
```
2. 异常
```shell
watch com.pupu.order.state.logistics.WaitConfirmLogisticsTaskState nextStep "{params, returnObj}" -e -x 2
```

#### 3.3. 代码是否被正确发布
- 判断方法是否存在
```she
sm com.pupu.order.state.logistics.WaitConfirmLogisticsTaskState nextStep
```

- 反编译代码
```shell
jad com.pupu.order.state.logistics.WaitConfirmLogisticsTaskState nextStep
ClassLoader:                                                                    
+-org.springframework.boot.loader.LaunchedURLClassLoader@21b8d17c               
  +-sun.misc.Launcher$AppClassLoader@677327b6                                   
    +-sun.misc.Launcher$ExtClassLoader@2ad3a1bb                                 

Location:                                                                       
file:/app/order-service.jar!/BOOT-INF/classes!/                                 

public LogisticsTasks nextStep(JwtUser jwtUser, String storeId, LogisticsTasks logisticsTask, String stageId) throws ExecutionException, InterruptedException {
    PupuAssert.isTrue((boolean)(logisticsTask.getLogisticsStatus().intValue() == LogisticsStatus.WAIT_CONFIRM.getValue()), (PupuResponsible)OrderErrorCode.NOT_WAIT_CONFIRM_STATUS);
    long timeNow = System.currentTimeMillis();
    LogisticsStatus targetLogisticsStatus = LogisticsStatus.CONFIRM;
    Orders order = this.logisticsTasksService.getOrders(logisticsTask.getSourceId());
    PupuAssert.notNull((Object)order, (PupuResponsible)OrderErrorCode.NOT_FOUND_ORDER);
    PupuAssert.hasText((String)order.getLogisticsTaskId(), (PupuResponsible)OrderErrorCode.EMPTY_LOGISTICS_TASK_ID);
    PupuAssert.isTrue((boolean)(!UUIDUtils.isEmptyUUID((String)order.getLogisticsTaskId())), (PupuResponsible)OrderErrorCode.EMPTY_LOGISTICS_TASK_ID);
    this.logisticsTasksService.getLogisticsTaskState(targetLogisticsStatus).updateOrderAttributes(jwtUser.getId(), jwtUser.getName(), order, timeNow, null, targetLogisticsStatus.getValue());
    AdminUsersDTO adminUsersDTO = this.logisticsTasksService.getAndValidateAdminUsersDTO(jwtUser.getId());
    LogisticsTasks modifiedLogisticsTask = this.logisticsTasksService.updateLogisticsTaskStatus(adminUsersDTO, adminUsersDTO, logisticsTask, order, targetLogisticsStatus, "\u65b0\u7248\u8c03\u5ea6\u7cfb\u7edf\uff1a\u624b\u52a8\u62a2\u5355", timeNow);
    return modifiedLogisticsTask;
}
```

#### 3.4. 跟踪代码执行路径以及耗时
```shell
trace com.pupu.order.state.logistics.WaitConfirmLogisticsTaskState nextStep -n 1
Press Q or Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 186 ms.
`---ts=2019-04-23 17:09:51;thread_name=http-nio-8080-exec-17;id=5c72;is_daemon=true;priority=5;TCCL=org.springframework.boot.context.embedded.tomcat.TomcatEmbeddedWebappClassLoader@5d42104a
    `---[13.712399ms] com.pupu.order.state.logistics.WaitConfirmLogisticsTaskState:nextStep()
        +---[0.043594ms] com.pupu.order.domain.LogisticsTasks:getLogisticsStatus()
        +---[0.004181ms] java.lang.Integer:intValue()
        +---[min=0.001294ms,max=0.003618ms,total=0.004912ms,count=2] com.pupu.core.common.enums.LogisticsStatus:getValue()
        +---[min=0.001192ms,max=0.006326ms,total=0.007518ms,count=2] com.pupu.core.utils.PupuAssert:isTrue()
        +---[0.009649ms] java.lang.System:currentTimeMillis()
        +---[0.003327ms] com.pupu.order.domain.LogisticsTasks:getSourceId()
        +---[0.507863ms] com.pupu.order.service.ILogisticsTasksService:getOrders()
        +---[0.003765ms] com.pupu.core.utils.PupuAssert:notNull()
        +---[min=0.001142ms,max=0.00692ms,total=0.008062ms,count=2] com.pupu.order.domain.Orders:getLogisticsTaskId()
        +---[0.006506ms] com.pupu.core.utils.PupuAssert:hasText()
        +---[0.005608ms] com.pupu.core.utils.UUIDUtils:isEmptyUUID()
        +---[0.004321ms] com.pupu.order.service.ILogisticsTasksService:getLogisticsTaskState()
        +---[min=0.001255ms,max=0.005872ms,total=0.007127ms,count=2] com.pupu.core.dto.web.JwtUser:getId()
        +---[0.002526ms] com.pupu.core.dto.web.JwtUser:getName()
        +---[0.001169ms] java.lang.Integer:<init>()
        +---[0.001286ms] java.lang.reflect.Method:invoke()
        +---[0.050409ms] com.pupu.order.state.logistics.ILogisticsTaskState:updateOrderAttributes()
        +---[2.796072ms] com.pupu.order.service.ILogisticsTasksService:getAndValidateAdminUsersDTO()
        `---[9.858717ms] com.pupu.order.service.ILogisticsTasksService:updateLogisticsTaskStatus()

Command execution times exceed limit: 1, so command will exit. You can set it with -n option.
```
trace com.pupu.marketing.service.cooking.CookingService update "{params, returnObj}" "params.length==2" -x 3

trace   com.pupu.marketing.service.cooking.CookingService update "params,returnObj" "params.length==2"  -x 3

#### 3.5. ognl条件表达式
- watch特定参数值
```shell
watch com.pupu.order.state.logistics.WaitConfirmLogisticsTaskState nextStep "{params, returnObj}" "params[1]=='6660fb8b-9a97-417c-ab47-1ef9c102e64a'" -x 3
Press Q or Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 150 ms.
ts=2019-04-23 17:00:40; [cost=13.303476ms] result=@ArrayList[
    @Object[][
        @JwtUser[
            id=@String[0c4f156b-d69a-4bf8-88cf-c164b07893e7],
            userName=@String[109266],
            name=@String[余秀禹],
            adminGroupId=@String[6bbb23c1-bc6d-41fb-bc9f-febb4dfeba34],
            isNotNovice=null,
        ],
        @String[6660fb8b-9a97-417c-ab47-1ef9c102e64a],
        @LogisticsTasks[
            id=@String[757a07e3-d4c1-4160-9f8b-061198694d9d],
            taskNum=@String[000101556009712984],
            storeId=@String[6660fb8b-9a97-417c-ab47-1ef9c102e64a],
            placeId=@String[6caac681-f336-4d26-aeb8-64c5e22c7162],
            storeDistance=@Double[2239.0],
            logisticsType=@Integer[10],
            sourceType=@Integer[0],
            sourceId=@String[757a07e3-d4c1-4160-9f8b-061198694d9d],
            sourceNum=@String[0101556009712984],
            buyerId=@String[146ef980-0057-4e0e-81bd-a7290793727e],
            logisticsStatus=@Integer[110],
            deliveryTimeType=@Integer[0],
            timeDeliveryPromise=@Long[1556011519437],
            isHasReceipt=@Boolean[false],
            executerId=@String[0c4f156b-d69a-4bf8-88cf-c164b07893e7],
            executerName=@String[余秀禹],
            executerPhone=@String[17605009769],
            taskStatus=@Integer[10],
            totalPrice=@Integer[1942],
            remark=@String[新版调度系统：手动抢单],
            userIdCreate=@String[00000000-0000-0000-0000-000000000000],
            timeCreate=@Long[1556009719437],
            timeUpdate=@Long[1556010040663],
            userIdUpdate=@String[00000000-0000-0000-0000-000000000000],
            receiverId=@String[7bd8be3d-e65b-42c6-bcb1-7174fc4a80e1],
            isPrint=@Boolean[true],
            processStatus=@Integer[10],
            internalTaskNum=@String[G1165567],
            timeDeliveryPromiseStart=@Long[1556011519437],
            timeDeliveryPromiseEnd=@Long[1556011519437],
            productWeight=@Integer[700],
            isReady=@Boolean[true],
            sortNum=@Integer[0],
        ],
        @String[],
    ],
    @LogisticsTasks[
        id=@String[757a07e3-d4c1-4160-9f8b-061198694d9d],
        taskNum=@String[000101556009712984],
        storeId=@String[6660fb8b-9a97-417c-ab47-1ef9c102e64a],
        placeId=@String[6caac681-f336-4d26-aeb8-64c5e22c7162],
        storeDistance=@Double[2239.0],
        logisticsType=@Integer[10],
        sourceType=@Integer[0],
        sourceId=@String[757a07e3-d4c1-4160-9f8b-061198694d9d],
        sourceNum=@String[0101556009712984],
        buyerId=@String[146ef980-0057-4e0e-81bd-a7290793727e],
        logisticsStatus=@Integer[110],
        deliveryTimeType=@Integer[0],
        timeDeliveryPromise=@Long[1556011519437],
        isHasReceipt=@Boolean[false],
        executerId=@String[0c4f156b-d69a-4bf8-88cf-c164b07893e7],
        executerName=@String[余秀禹],
        executerPhone=@String[17605009769],
        taskStatus=@Integer[10],
        totalPrice=@Integer[1942],
        remark=@String[新版调度系统：手动抢单],
        userIdCreate=@String[00000000-0000-0000-0000-000000000000],
        timeCreate=@Long[1556009719437],
        timeUpdate=@Long[1556010040663],
        userIdUpdate=@String[00000000-0000-0000-0000-000000000000],
        receiverId=@String[7bd8be3d-e65b-42c6-bcb1-7174fc4a80e1],
        isPrint=@Boolean[true],
        processStatus=@Integer[10],
        internalTaskNum=@String[G1165567],
        timeDeliveryPromiseStart=@Long[1556011519437],
        timeDeliveryPromiseEnd=@Long[1556011519437],
        productWeight=@Integer[700],
        isReady=@Boolean[true],
        sortNum=@Integer[0],
    ],
]
```

- 关注执行慢的函数
```shell
trace com.pupu.order.state.logistics.WaitConfirmLogisticsTaskState nextStep '#cost > 10'
Press Q or Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 167 ms.
`---ts=2019-04-23 17:07:49;thread_name=http-nio-8080-exec-31;id=5c91;is_daemon=true;priority=5;TCCL=org.springframework.boot.context.embedded.tomcat.TomcatEmbeddedWebappClassLoader@5d42104a
    `---[13.41925ms] com.pupu.order.state.logistics.WaitConfirmLogisticsTaskState:nextStep()
        +---[0.019738ms] com.pupu.order.domain.LogisticsTasks:getLogisticsStatus()
        +---[0.004759ms] java.lang.Integer:intValue()
        +---[min=0.004369ms,max=0.020002ms,total=0.024371ms,count=2] com.pupu.core.common.enums.LogisticsStatus:getValue()
        +---[min=0.00206ms,max=0.005935ms,total=0.007995ms,count=2] com.pupu.core.utils.PupuAssert:isTrue()
        +---[0.010221ms] java.lang.System:currentTimeMillis()
        +---[0.003777ms] com.pupu.order.domain.LogisticsTasks:getSourceId()
        +---[0.370217ms] com.pupu.order.service.ILogisticsTasksService:getOrders()
        +---[0.005365ms] com.pupu.core.utils.PupuAssert:notNull()
        +---[min=0.001964ms,max=0.008411ms,total=0.010375ms,count=2] com.pupu.order.domain.Orders:getLogisticsTaskId()
        +---[0.008749ms] com.pupu.core.utils.PupuAssert:hasText()
        +---[0.007843ms] com.pupu.core.utils.UUIDUtils:isEmptyUUID()
        +---[0.005402ms] com.pupu.order.service.ILogisticsTasksService:getLogisticsTaskState()
        +---[min=0.002019ms,max=0.007799ms,total=0.009818ms,count=2] com.pupu.core.dto.web.JwtUser:getId()
        +---[0.271536ms] com.pupu.core.dto.web.JwtUser:getName()
        +---[0.054686ms] com.pupu.order.state.logistics.ILogisticsTaskState:updateOrderAttributes()
        +---[3.046402ms] com.pupu.order.service.ILogisticsTasksService:getAndValidateAdminUsersDTO()
        `---[8.517051ms] com.pupu.order.service.ILogisticsTasksService:updateLogisticsTaskStatus()
```

#### 3.6. 找出cpu占用高的线程并打印stack trace信息
- thread查看全局线程cpu占用情况
```shell
$ thread
Threads Total: 441, NEW: 0, RUNNABLE: 79, BLOCKED: 0, WAITING: 311, TIMED_WAITIN
G: 51, TERMINATED: 0                                                            
ID     NAME                GROUP        PRIORI STATE %CPU   TIME   INTER DAEMON 
225802 as-command-execute- system       10     RUNNA 66     0:0    false true   
427    DubboServerHandler- main         5      WAITI 14     0:4    false true   
159    Thread-10           main         5      TIMED 4      3:8    false false  
122    New I/O server work main         5      RUNNA 3      0:35   false true   
100446 main-SendThread(172 main         5      RUNNA 2      0:0    false true   
198    DubboResponseTimeou main         5      TIMED 1      1:7    false true   
40     redisson-netty-1-3  main         5      RUNNA 1      0:2    false false  
211    AMQP Connection 172 main         5      RUNNA 0      0:4    false false  
36     Abandoned connectio main         5      TIMED 0      0:22   false true 
```

- thread -n 查看排名靠前的线程
```shell
$ thread -n 3
"as-command-execute-daemon" Id=226330 cpuUsage=75% RUNNABLE
    at sun.management.ThreadImpl.dumpThreads0(Native Method)
    at sun.management.ThreadImpl.getThreadInfo(ThreadImpl.java:448)
    at com.taobao.arthas.core.command.monitor200.ThreadCommand.processTopBusyThreads(ThreadCommand.java:133)
    at com.taobao.arthas.core.command.monitor200.ThreadCommand.process(ThreadCommand.java:79)
    at com.taobao.arthas.core.shell.command.impl.AnnotatedCommandImpl.process(AnnotatedCommandImpl.java:82)
    at com.taobao.arthas.core.shell.command.impl.AnnotatedCommandImpl.access$100(AnnotatedCommandImpl.java:18)
    at com.taobao.arthas.core.shell.command.impl.AnnotatedCommandImpl$ProcessHandler.handle(AnnotatedCommandImpl.java:111)
    at com.taobao.arthas.core.shell.command.impl.AnnotatedCommandImpl$ProcessHandler.handle(AnnotatedCommandImpl.java:108)
    at com.taobao.arthas.core.shell.system.impl.ProcessImpl$CommandProcessTask.run(ProcessImpl.java:370)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
    at java.lang.Thread.run(Thread.java:748)

    Number of locked synchronizers = 1
    - java.util.concurrent.ThreadPoolExecutor$Worker@4389bd2b

"Thread-10" Id=159 cpuUsage=5% TIMED_WAITING
    at java.lang.Thread.sleep(Native Method)
    at com.umpay.bank.Sender.run(Sender.java:49)

"DubboClientReconnectTimer-thread-2" Id=101 cpuUsage=2% TIMED_WAITING on java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject@369165d5
    at sun.misc.Unsafe.park(Native Method)
    -  waiting on java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject@369165d5
    at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:215)
    at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.awaitNanos(AbstractQueuedSynchronizer.java:2078)
    at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:1093)
    at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:809)
    at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1074)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1134)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
    at java.lang.Thread.run(Thread.java:748)
```

- thread {threadId} 查看线程栈信息
```shell
$ thread 101
"DubboClientReconnectTimer-thread-2" Id=101 TIMED_WAITING on java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject@369165d5
    at sun.misc.Unsafe.park(Native Method)
    -  waiting on java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject@369165d5
    at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:215)
    at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.awaitNanos(AbstractQueuedSynchronizer.java:2078)
    at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:1093)
    at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:809)
    at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1074)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1134)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
    at java.lang.Thread.run(Thread.java:748)

Affect(row-cnt:0) cost in 80 ms.
```

#### 3.7. 方法重载如何处理
- 找出所有的重载方法
```shell
sm com.pupu.order.repository.CommentRepository listKnightComment
com.sun.proxy.$Proxy200 listKnightComment(JJ)Ljava/util/List;
com.sun.proxy.$Proxy200 listKnightComment(Ljava/lang/String;JJ)Ljava/util/List;
com.pupu.order.repository.CommentRepository listKnightComment(JJ)Ljava/util/List;
com.pupu.order.repository.CommentRepository listKnightComment(Ljava/lang/String;JJ)Ljava/util/List;
Affect(row-cnt:4) cost in 21 ms.
```
- ognl表达式过滤
```shell
watch com.pupu.order.repository.CommentRepository listKnightComment "{params, returnObj}" "params.length > 3" -x 2
```

#### 3.8. Arthas端口冲突和日志查看
有的时候watch或者sm等命令提示未找到类，可能是端口冲突引起的

1. 查看Arthas日志
```shell
tail -100 /root/logs/arthas/
01 2019-04-23 16:40:47.893 ERROR [arthas-binding-thread:arthas] [] [] [] Error listening to port 8563
java.net.BindException: Address already in use
	at sun.nio.ch.Net.bind0(Native Method) ~[na:1.8.0_181]
	at sun.nio.ch.Net.bind(Net.java:433) ~[na:1.8.0_181]
	at sun.nio.ch.Net.bind(Net.java:425) ~[na:1.8.0_181]
	at sun.nio.ch.ServerSocketChannelImpl.bind(ServerSocketChannelImpl.java:223) ~[na:1.8.0_181]
	at io.netty.channel.socket.nio.NioServerSocketChannel.doBind(NioServerSocketChannel.java:130) ~[arthas-core.jar:3.1.0]
	at io.netty.channel.AbstractChannel$AbstractUnsafe.bind(AbstractChannel.java:558) ~[arthas-core.jar:3.1.0]
	at io.netty.channel.DefaultChannelPipeline$HeadContext.bind(DefaultChannelPipeline.java:1358) ~[arthas-core.jar:3.1.0]
	at io.netty.channel.AbstractChannelHandlerContext.invokeBind(AbstractChannelHandlerContext.java:501) ~[arthas-core.jar:3.1.0]
	at io.netty.channel.AbstractChannelHandlerContext.bind(AbstractChannelHandlerContext.java:486) ~[arthas-core.jar:3.1.0]
	at io.netty.handler.logging.LoggingHandler.bind(LoggingHandler.java:191) ~[arthas-core.jar:3.1.0]
	at io.netty.channel.AbstractChannelHandlerContext.invokeBind(AbstractChannelHandlerContext.java:501) ~[arthas-core.jar:3.1.0]
	at io.netty.channel.AbstractChannelHandlerContext.bind(AbstractChannelHandlerContext.java:486) ~[arthas-core.jar:3.1.0]
	at io.netty.channel.DefaultChannelPipeline.bind(DefaultChannelPipeline.java:1019) ~[arthas-core.jar:3.1.0]
	at io.netty.channel.AbstractChannel.bind(AbstractChannel.java:254) ~[arthas-core.jar:3.1.0]
	at io.netty.bootstrap.AbstractBootstrap$2.run(AbstractBootstrap.java:366) ~[arthas-core.jar:3.1.0]
	at io.netty.util.concurrent.AbstractEventExecutor.safeExecute(AbstractEventExecutor.java:163) ~[arthas-core.jar:3.1.0]
	at io.netty.util.concurrent.SingleThreadEventExecutor.runAllTasks(SingleThreadEventExecutor.java:404) ~[arthas-core.jar:3.1.0]
	at io.netty.channel.nio.NioEventLoop.run(NioEventLoop.java:446) ~[arthas-core.jar:3.1.0]
	at io.netty.util.concurrent.SingleThreadEventExecutor$5.run(SingleThreadEventExecutor.java:884) ~[arthas-core.jar:3.1.0]
	at io.netty.util.concurrent.FastThreadLocalRunnable.run(FastThreadLocalRunnable.java:30) ~[arthas-core.jar:3.1.0]
	at java.lang.Thread.run(Thread.java:748) ~[na:1.8.0_181]
```
2. 找到被占用端口的进程
```shell
netstat -tlunp | egrep "3658|8563"
```
3. 找到Docker进程
```shell
ps aux| grep 86308
```
4. 进入Docker
```shell
docker exec -it inapideploy_order-service_1 bash
```
5. shutdown Docker内存的Arthas
```shell
java -jar /app/arthas-boot.jar 1 -c 'shutdown'


```