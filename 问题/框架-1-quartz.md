# Quartz

## 依赖

````xml
        <!--quartz-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-quartz</artifactId>
        </dependency>
````

## 配置

````yaml
spring:
  # quartz
  quartz:
    properties:
      org:
        quartz:
          scheduler:
            instanceId: AUTO
            instanceName: schedulerFactory
            wrapJobExecutionInUserTransaction: false
            rmi:
              export: false
              proxy: false
          # 实例化ThreadPool时，使用的线程类为SimpleThreadPool
          threadPool:
            class: org.quartz.simpl.SimpleThreadPool
            #并发个数
            threadCount: 15
            # 优先级
            threadPriority: 5
            threadsInheritContextClassLoaderOfInitializingThread: true
          jobStore:
            #最大能忍受的触发超时时间
            misfireThreshold: 180000
            #持久化
            class: org.springframework.scheduling.quartz.LocalDataSourceJobStore
            driverDelegateClass: org.quartz.impl.jdbcjobstore.StdJDBCDelegate
            tablePrefix: qrtz_
            isClustered: false
            clusterCheckinInterval: 180000
    job-store-type: jdbc
    jdbc:
      initialize-schema: never 
````

## Job

我们想要调度的任务都必须实现 `org.quartz.job` 接口，然后实现接口中定义的 `execute( )` 方法

***`JobDataMap`，JobDetail对象的一部分***。

````java
@Slf4j
public class HelloJob implements Job {

    @Override
    public void execute(JobExecutionContext context) {
        log.info("Hello Job 执行时间: {}", DateUtil.now());
    }
}

````

````java
public class HelloJob implements Job {
    public HelloJob() {
    }

    public void execute(JobExecutionContext context)throws JobExecutionException{
      JobKey key = context.getJobDetail().getKey();
      JobDataMap dataMap = context.getJobDetail().getJobDataMap();
        
      String jobSays = dataMap.getString("jobSays");
      float myFloatValue = dataMap.getFloat("myFloatValue");
      System.err.println("Instance " + key + " of HelloJob says: " + jobSays + ", and val is: " + myFloatValue);
    }
}

````

| 无状态的 Job | 每次调用时都会创建一个新的 JobDataMap                        |
| ------------ | ------------------------------------------------------------ |
| 有状态的 Job | 多次 Job 调用可以持有一些状态信息，这些状态信息存储在 JobDataMap 中 |

***跟踪 Job状态时***，需要在任务类上加个注解`@PersistJobDataAfterExecution`，让 Job 变成有状态

````java
@Slf4j
@PersistJobDataAfterExecution
public class HelloJob implements Job {

    private Integer executeCount;

    public void setExecuteCount(Integer executeCount) {
        this.executeCount = executeCount;
    }

    @Override
    public void execute(JobExecutionContext jobExecutionContext) throws JobExecutionException {
        String data = LocalDateTime.now().
            format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
        log.info("execute count: {}, current time: {}",
                 ++executeCount, data);
        //将累加的 count 存入JobDataMap中
        jobExecutionContext.getJobDetail().
            getJobDataMap().put("executeCount", executeCount);
    }
}
/** OUTPUT:
execute count: 1, current time: 2020-11-17 22:28:48
execute count: 2, current time: 2020-11-17 22:28:52
execute count: 3, current time: 2020-11-17 22:28:57
**/

````

## Trigger

- SimpleTrigger在一个指定时间段内执行一次作业任务或是在指定时间间隔内执行多次作业任务CronTrigger基于日历的作业调度器，
- Cron表达式配置CronTrigger的实例，而不是像SimpleTrigger那样精确指定间隔时间，比SimpleTrigger更常用

![image-20241031201000530](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202410312010643.png)

## Entity

````java
````

