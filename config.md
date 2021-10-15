
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
