
# 1.引入相应的pom

    <dependency>
  
      <groupId>com.dangdang</groupId>
      <artifactId>elastic-job-lite-spring</artifactId>
      <version>2.1.5</version>
      
    </dependency>
   
    <dependency>
    
      <groupId>com.dangdang</groupId>
      <artifactId>elastic-job-lite-core</artifactId>
      <version>2.1.5</version>
      
    </dependency>
    
# 2.配置zk，定时任务属性

@Configuration

public class ElasticJobConfig {

    @Value("${job.cron}")
    private String jobCron;

    @Value("${expire.cron}")
    private String expireCron;

    @Value("${zk.servers}")
    private String serverList;

    @Value("${regcenter.namespace}")
    private String namespace;

    @Autowired 
    private ZookeeperRegistryCenter regCenter;

    @Bean(initMethod = "init")
    public ZookeeperRegistryCenter regCenter() {
        return new ZookeeperRegistryCenter(
                new ZookeeperConfiguration(serverList, namespace));
    }

    private LiteJobConfiguration getLiteJobConfiguration(
            final Class<? extends SimpleJob> jobClass, final String cron,
            final int shardingTotalCount, final String shardingItemParameters) {
        return LiteJobConfiguration.newBuilder(new SimpleJobConfiguration(
                JobCoreConfiguration
                        .newBuilder(jobClass.getName(), cron, shardingTotalCount)
                        .shardingItemParameters(shardingItemParameters).build(),
                jobClass.getCanonicalName())).overwrite(true).build();
    }

    //若有多个任务，可配置多个bean，用于配置已知的定时任务
    @Bean(initMethod = "init")
    public JobScheduler simpleJobScheduler(XXXJob xxxJob) {
    
        //此处配置为一个分片，无需分片参数
        return new SpringJobScheduler(xxxJob, regCenter,
                                      getLiteJobConfiguration(xxxJob.getClass(),
                                                              jobCron, 1, null),
                                      new ElasticJobListener() {
                                          @Override
                                          public void beforeJobExecuted(
                                                  ShardingContexts shardingContexts) {

                                          }

                                          @Override
                                          public void afterJobExecuted(
                                                  ShardingContexts shardingContexts) {

                                          }
                                      });
    }
    
    private LiteJobConfiguration getLiteJobConfiguration(
            final Class<? extends SimpleJob> jobClass, final String cron,
            final int shardingTotalCount, final String shardingItemParameters) {
        return LiteJobConfiguration.newBuilder(new SimpleJobConfiguration(
                JobCoreConfiguration
                        .newBuilder(jobClass.getName(), cron, shardingTotalCount)
                        .shardingItemParameters(shardingItemParameters).build(),
                jobClass.getCanonicalName())).overwrite(true).build();
    }
    
    //配置动态添加定时任务
  
     /**
     * 添加定时任务
     */
    public void addJob(SimpleJob job,String cron,Integer shardCount,String id){
        new SpringJobScheduler(job, regCenter,
                               getLiteJobConfiguration(job.getClass(), cron, shardCount,
                                                       null), new AbstractDistributeOnceElasticJobListener(100, 100) {
            @Override
            public void doBeforeJobExecutedAtLastStarted(
                    ShardingContexts shardingContexts) {

            }

            @Override
            public void doAfterJobExecutedAtLastCompleted(
                    ShardingContexts shardingContexts) {

            }
        });
    }

    /**
     * 更新定时任务
     */

    public void updateJob(SimpleJob job ,String cron){
        JobRegistry.getInstance().getJobScheduleController(job.getClass().getName()).rescheduleJob(cron);
    }

    /**
     * 删除定时任务
     */

    public void removeJob(SimpleJob job) {
        try {
            JobRegistry.getInstance().getJobScheduleController(job.getClass().getName()).shutdown();
            
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

    
    
# 3.定义任务类

@Slf4j

@Component

public class XXXJob implements SimpleJob {

    @Override
    public void execute(ShardingContext shardingContext) {
        //具体业务逻辑
    }
  }
  
# 4.原理解析 elastic-job-lite
     elastic-job-lite，轻量级无中心化解决方案，使用jar包的形式提供分布式的协调服务。
     核心组件:quartz、zookeeper
## 4.1 zookeeper
     1. 使用zk作为注册中心，作业实例信息、分片信息、配置信息、作业运行状态等均以节点的方式存于zk。
     2. zk上目录:/namespace/jobName/(leader,servers,config,instances,sharding)，其中
        leader：包括(election,sharding)，存放主节点选举信息、重分片信息，
        servers：服务节点的ip地址；
        instances：服务节点的ip地址@-@进程号；
        config： 初始化作业时com.dangdang.ddframe.job.lite.api.JobScheduler#init，存储作业配置信息com.dangdang.ddframe.job.lite.config.LiteJobConfiguration，如下所示：
                {
                     "jobName": "xxxx",
                     "jobClass": "xxx",
                     "jobType": "SIMPLE",
                     "cron": "0 58 23 * * ? ",
                     "shardingTotalCount": 1,
                     "shardingItemParameters": "",
                     "jobParameter": "",
                     "failover": false,
                     "misfire": true,
                     "description": "",
                     "jobProperties": {
                             "job_exception_handler": "com.dangdang.ddframe.job.executor.handler.impl.DefaultJobExceptionHandler",
                             "executor_service_handler": "com.dangdang.ddframe.job.executor.handler.impl.DefaultExecutorServiceHandler"
                       },
                   "monitorExecution": true,
                   "maxTimeDiffSeconds": -1,
                   "monitorPort": -1,
                   "jobShardingStrategyClass": "",
                   "reconcileIntervalMinutes": 10,
                   "disabled": false,
                   "overwrite": true
             }
     3. 通过zk的节点变更事件完成分布式任务协同，节点新增、变更、移除等事件会实时同步给分布式环境中的每个作业实例，并提供了多种监听器来处理这些事件，
        其中，监听器父类AbstractJobListener以及TreeCacheListener:
        electionListenerManager：当leader节点被删除或者leader节点不存在时;如果本身节点被置为不可用并且本身节点是leader，则移除本身节点的leader节点，触发leaderService.electLeader选举。
        shardingListenerManager：配置节点的分片总数发生改变，server或者instance节点发生改变，设置重新分片标识(新增/namespace/jobName/leader/sharding/necerrary持久节点)，
                                强制任务执行重新分片；
        failoverListenerManager：instance节点下线，将其拥有的失效转移任务再次触发失效转移到其他节点执行；配置节点中的isFailover发生改变，若不需要失效转移，则直接清除现有的失效转移任务
        monitorExecutionListenerManager：配置节点中的isMonitorExecution发生改变，若不需要提醒执行信息，则直接清除现有的显示执行的节点。
        shutdownListenerManager：本身的instance节点被删除,需要关闭本身节点的定时任务；
        triggerListenerManager：出现指定本身节点的trigger节点，将清除trigger节点，然后调用triggerJob立刻执行一次任务。
        rescheduleListenerManager：配置节点中的cron发生改变，调用rescheduleJob重新调度定时任务。
        guaranteeListenerManager：start节点发生改变，通知任务启动；complete节点发生改变，通知任务完成；
        regCenterConnectionStateListener：监听zookeeper的连接状态，若连接出现挂起或者丢失状态，则暂停任务，
                                          如果重新连接上，则重新写入server、instance本身节点到zookeeper，清除本身节点的运行节点，然后重新调度定时任务；
     4. 主节点的选举：利用zk的分布式锁机制，所有节点都往同一路径下创建顺序节点，只有获取到最小序号的节点会执行LeaderElectionExecutionCallback的execute方法，将本身节点的信息写入leader节                      点(/leader/election/instance)，后续节点再次进入发现leader节点拥有数据，则自己只能成为follower节点结束选举流程。
  ## 4.2 quartz
        1. 初始化时，将elasticjob和jobFacade这两个key放到quartz的JobDetail#JobDataMap中
           作业实例内任务触发是通过Quartz来完成的，按照cron设定的时间定时触发，触发Job的实现LiteJob#execute方法，
           调用com.dangdang.ddframe.job.executor.AbstractElasticJobExecutor#execute()方法
           
        2. 从zk中获取作业的相关信息，/config中获取作业配置信息，/sharding/{i}/instance 获取运行在本作业实例的分片项集合，i表示分片项，
        通过com.dangdang.ddframe.job.lite.internal.schedule.LiteJobFacade#getShardingContexts，获取当前作业服务器分片上下文。
        
        3. 如果是单片分，当前作业实例只有一个分片，则直接在原线程执行任务；如果是多分片情况，采用线程池机制，一个分片一个线程，然后使用countDownLatch等待所有任务执行完成。
        
        4. AbstractElasticJobExecutor#process(com.dangdang.ddframe.job.api.ShardingContext)三种job形态：
           SimpleJobExecutor：直接调用用户定义的SimpleJob中的execute()方法执行即可；
           ScriptJobExecutor：将分片参数转化成命令行，调用底层的命令行执行方法即可；
           DataflowJobExecutor：根据配置选择对应的流式处理模式，如果是oneOffExecute，则调用一次fetchData拉取数据，再调用一次processData处理数据就结束；
                                如果是streamingExecute，则会不断进行拉取处理的循环，直到拉取的数据为空。


 
     
        
        
        
        
        
        
        
        
        
     
