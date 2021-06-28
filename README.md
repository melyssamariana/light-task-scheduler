
# LTS user documentation (NOT THE ORIGINAL PROJEQT)
	
LTS (light-task-scheduler) is mainly used to solve distributed task scheduling problems, supporting real-time tasks, timing tasks and cron tasks. It has good scalability, scalability, robustness and stability, and is used by many companies. At the same time, we also hope that open source enthusiasts can contribute together.

## ---> There are recruitment posts at the bottom

## project address
github address:
[https://github.com/ltsopensource/light-task-scheduler](https://github.com/ltsopensource/light-task-scheduler)

oschina address:
[http://git.oschina.net/hugui/light-task-scheduler](http://git.oschina.net/hugui/light-task-scheduler)

Example:
[https://github.com/ltsopensource/lts-examples](https://github.com/ltsopensource/lts-examples)

Document address (under update, please refer to this later):
[https://www.gitbook.com/book/qq254963746/lts/details](https://www.gitbook.com/book/qq254963746/lts/details)

Both addresses will be updated simultaneously. If you are interested, please add QQ group: 109500214 (add group password: hello world) to discuss and improve together. The more people support, the more motivated to update. I like to remember the star in the upper right corner.

## 1.7.2-SNAPSHOT(master) changes main points
1. Optimize BizLogger in JobContext, remove threadlocal from the original, solve the problem of taskTracker multi-threading, remove LtsLoggerFactory.getLogger() usage

## Framework Overview
LTS has the following four main types of nodes:

* JobClient: Mainly responsible for submitting tasks, and receiving task execution feedback results.
* JobTracker: Responsible for receiving and assigning tasks, task scheduling.
* TaskTracker: Responsible for executing tasks, and feedback to JobTracker after execution.
* LTS-Admin: (Administration background) is mainly responsible for node management, task queue management, monitoring management, etc.

Among them, JobClient, JobTracker, TaskTracker nodes are all `stateless`.
Multiple deployments can be deployed and deleted dynamically to achieve load balancing and achieve greater load, and the framework adopts the FailStore strategy to make LTS have good fault tolerance.

The LTS registry provides multiple implementations (Zookeeper, redis, etc.). The registry performs node information exposure and master election. (Mongo or Mysql) Store task queues and task execution logs, netty or mina for underlying communication, and provide multiple serialization methods fastjson, hessian2, java, etc.

LTS supports task types:

* Real-time tasks: tasks that will be executed immediately after submission.
* Timing task: a task executed at a specified time point, for example, executed at 3 o'clock today (single time).
* Cron task: CronExpression, similar to quartz (but not implemented using quartz) such as 0 0/1 * * *?

Support dynamic modification of task parameters, task execution time and other settings, support dynamic addition of tasks in the background, support Cron task suspension, support manual stop of tasks being executed (with conditions), support task monitoring statistics, support task execution monitoring of each node, JVM Monitoring and more.

## Architecture diagram

![LTS architecture](http://git.oschina.net/hugui/light-task-scheduler/raw/master/docs/LTS_architecture.png?dir=0&filepath=docs%2FLTS_architecture.png&oid=262a5234534e2d9fa8862f3e632c5551ebd95e21&sha=d01be5d59e8d768f49bbdc66c8334c37af8f7af5)

## Concept description

### Node Group
1. English name NodeGroup, a node group is equivalent to a small cluster, each node in the same node group is equal, equivalent, and provides the same service to the outside.
2. Each node group has a master node. This master node is dynamically selected by LTS. When a master node goes down, LTS will immediately select another master node. The framework provides an API monitoring interface for users.

### FailStore
1. As the name suggests, this is mainly used for failed storage, mainly for node fault tolerance. When the remote data interaction fails, it is stored locally, and the data is submitted again when the remote communication is restored.
2. FailStore's main user JobClient's task submission, TaskTracker's task feedback, TaskTracker's business log transmission scenarios.
3. FailStore currently provides several implementations: leveldb, rocksdb, berkeleydb, mapdb, ltsdb, which can be used freely, and users can also use SPI extensions to use their own implementation.


## Flow chart
The figure below is a standard real-time task execution process.

![LTS progress](http://git.oschina.net/hugui/light-task-scheduler/raw/master/docs/LTS_progress.png?dir=0&filepath=docs%2FLTS_progress.png&oid=22f60a83b51b26bac8dabbb5053ec9913cebbfc45c&sha=774aa73d44702becbba886)

## LTS-Admin new version interface preview

![LTS Admin](http://git.oschina.net/hugui/light-task-scheduler/raw/master/docs/LTS-Admin/LTS-Admin-cron-job-queue.png?dir=0&filepath= docs%2FLTS-Admin%2FLTS-Admin-cron-job-queue.png&oid=aecaf01bca5270a53b144891baaa3d7e56d47706&sha=a4fd9f31df9e1fc6d389a16bdc8d1964bb854766)
At present, the background has a simple authentication function provided by [ztajy](https://github.com/ztajy). The user name and password are in auth.cfg, and the user can modify it by himself.

## Features
### 1. Spring support
LTS can completely use the Spring framework, but considering that the Spring framework is used in many user projects, LTS also provides support for Spring, including Xml and annotations, just import `lts-spring.jar`.
### 2. Business Log Recorder
A business log recorder is provided on the TaskTracker side, which is used by application programs. Through this business logger, business logs can be submitted to JobTracker. These business logs can be connected by task ID, and the execution of tasks can be viewed in LTS-Admin in real time schedule.
### 3. SPI extension support
SPI extension can achieve zero intrusion, only need to implement the corresponding interface, and the implementation can be used by LTS, currently open extended interfaces are

1. To extend the task queue, users can choose not to use mysql or mongo as queue storage, or they can implement it themselves.
2. The expansion of the business log recorder currently mainly supports console, mysql, mongo, and users can also choose to send logs to other places through the expansion.

### 4. Failover
When the TaskTracker that is executing the task goes down, the JobTracker will immediately allocate all the tasks assigned to the down TaskTracker to other normal TaskTracker nodes for execution.
### 5. Node monitoring
You can perform resource monitoring and task monitoring on JobTracker and TaskTracker nodes, and you can view them in the LTS-Admin management background in real time, and then perform reasonable resource allocation.
### 6, diversified task execution results support
The LTS framework provides four kinds of execution result support, `EXECUTE_SUCCESS`, `EXECUTE_FAILED`, `EXECUTE_LATER`, `EXECUTE_EXCEPTION`, and adopts corresponding processing mechanism for each result, such as retry.

* EXECUTE_SUCCESS: The execution is successful. In this case, it will be directly reported back to the client (if the task is set, it will be reported back to the client).
* EXECUTE_FAILED: The execution fails. In this case, it is directly reported to the client without retrying.
* EXECUTE_LATER: Execute later (retry required). In this case, the client is not fed back. The retry strategy adopts 1min, 2min, and 3min strategies. The default maximum number of retries is 10 times. The user can modify this retry through parameter settings. Number of attempts.
* EXECUTE_EXCEPTION: Execution is abnormal, this situation will also be retried (retry strategy, the same as above)上)

### 7, FailStore fault tolerance
The FailStore mechanism is used for node fault tolerance. Fail And Store will not affect the operation of the current application due to the instability of remote communication. For specific FailStore description, please refer to the FailStore description in the concept description.

## Project compilation and packaging
The project is mainly built using maven, and currently provides shell script packaging.
Environment dependency: `Java(jdk1.6+)` `Maven`

User usage is generally divided into two types:
### 1. Maven build
The jar package of lts can be uploaded to the local warehouse through the maven command. Add the corresponding repository in the parent pom.xml and upload it with the deploy command. For specific citation methods, please refer to the example in lts.
### 2. Direct Jar reference
Each module of lts needs to be packaged into a separate jar package, and all lts dependent packages are introduced. You can refer to the examples in lts for specific jar packages to be cited.

## JobTracker and LTS-Admin deployment
Two scripts of `(cmd)windows` and `(shell)linux` are provided for compilation and deployment:

1. Run the `sh build.sh` or `build.cmd` script in the root directory, and the `lts-{version}-bin` folder will be generated in the `dist` directory

2. The following is its directory structure, where the bin directory is mainly the startup script of JobTracker and LTS-Admin. `jobtracker` is the configuration file of JobTracker and the jar package that needs to be used, `lts-admin` is the war package and configuration file related to LTS-Admin.
File structure of lts-{version}-bin

```java
-- lts-${version}-bin
    |-- bin
    |   |-- jobtracker.cmd
    |   |-- jobtracker.sh
    |   |-- lts-admin.cmd
    |   |-- lts-admin.sh
    |   |-- lts-monitor.cmd
    |   |-- lts-monitor.sh
    |   |-- tasktracker.sh
    |-- conf
    |   |-- log4j.properties
    |   |-- lts-admin.cfg
    |   |-- lts-monitor.cfg
    |   |-- readme.txt
    |   |-- tasktracker.cfg
    |   |-- zoo
    |       |-- jobtracker.cfg
    |       |-- log4j.properties
    |       |-- lts-monitor.cfg
    |-- lib
    |   |-- *.jar
    |-- war
        |-- jetty
        |   |-- lib
        |       |-- *.jar
        |-- lts-admin.war

```	    
        
3. JobTracker starts. If you want to start a node, directly modify the configuration file under `conf/zoo`, and then run `sh jobtracker.sh zoo start`. If you want to start two JobTracker nodes, then you need to copy a zoo, For example, name it as `zoo2`, modify the configuration file under `zoo2`, and then run `sh jobtracker.sh zoo2 start`. The `jobtracker-zoo.out` log is generated under the logs folder.
4. Start LTS-Admin. Modify the configuration under `conf/lts-monitor.cfg` and `conf/lts-admin.cfg`, and then run `sh lts-admin.sh` or `lts- under `bin` The admin.cmd` script is fine. The `lts-admin.out` log will be generated under the logs folder, and the access address will be printed out in the log if the startup is successful, and the user can access through this access address.

## JobClient (deployment) use
The jar packages that need to import lts include `lts-jobclient-{version}.jar`, `lts-core-{version}.jar` and other third-party dependent jars.
### API start
```java
JobClient jobClient = new RetryJobClient();
jobClient.setNodeGroup("test_jobClient");
jobClient.setClusterName("test_cluster");
jobClient.setRegistryAddress("zookeeper://127.0.0.1:2181");
jobClient.start();

// Submit task
Job job = new Job();
job.setTaskId("3213213123");
job.setParam("shopId", "11111");
job.setTaskTrackerNodeGroup("test_trade_TaskTracker");
// job.setCronExpression("0 0/1 * * * ?"); // support cronExpression expression
// job.setTriggerTime(new Date()); // Support execution at specified time
Response response = jobClient.submitJob(job);
```
    
### Spring XML start
```java
<bean id="jobClient" class="com.github.ltsopensource.spring.JobClientFactoryBean">
    <property name="clusterName" value="test_cluster"/>
    <property name="registryAddress" value="zookeeper://127.0.0.1:2181"/>
    <property name="nodeGroup" value="test_jobClient"/>
    <property name="masterChangeListeners">
        <list>
            <bean class="com.github.ltsopensource.example.support.MasterChangeListenerImpl"/>
        </list>
    </property>
    <property name="jobFinishedHandler">
        <bean class="com.github.ltsopensource.example.support.JobFinishedHandlerImpl"/>
    </property>
    <property name="configs">
        <props>
           <!-- Parameters -->
            <prop key="job.fail.store">leveldb</prop>
        </props>
    </property>
</bean>
```    
### Spring full annotation method
```java
@Configuration
public class LTSSpringConfig {

    @Bean(name = "jobClient")
    public JobClient getJobClient() throws Exception {
        JobClientFactoryBean factoryBean = new JobClientFactoryBean();
        factoryBean.setClusterName("test_cluster");
        factoryBean.setRegistryAddress("zookeeper://127.0.0.1:2181");
        factoryBean.setNodeGroup("test_jobClient");
        factoryBean.setMasterChangeListeners(new MasterChangeListener[]{
                new MasterChangeListenerImpl()
        });
        Properties configs = new Properties();
        configs.setProperty("job.fail.store", "leveldb");
        factoryBean.setConfigs(configs);
        factoryBean.afterPropertiesSet();
        return factoryBean.getObject();
    }
}
```
## TaskTracker (for deployment)
The jar packages that need to import lts include `lts-tasktracker-{version}.jar`, `lts-core-{version}.jar` and other third-party dependent jars.
### Define your own task execution class
```java
public class MyJobRunner implements JobRunner {
    @Override
    public Result run(JobContext jobContext) throws Throwable {
        try {
             // TODO business logic
             // will be sent to LTS (on JobTracker)
             jobContext.getBizLogger().info("Test, business log ah ah ah ah");

        } catch (Exception e) {
            return new Result(Action.EXECUTE_FAILED, e.getMessage());
        }
       return new Result(Action.EXECUTE_SUCCESS, "The execution was successful, haha");
    }
}
```
### API start
```java 
TaskTracker taskTracker = new TaskTracker();
taskTracker.setJobRunnerClass(MyJobRunner.class);
taskTracker.setRegistryAddress("zookeeper://127.0.0.1:2181");
taskTracker.setNodeGroup("test_trade_TaskTracker");
taskTracker.setClusterName("test_cluster");
taskTracker.setWorkThreads(20);
taskTracker.start();
```
### Spring XML start
```java
<bean id="taskTracker" class="com.github.ltsopensource.spring.TaskTrackerAnnotationFactoryBean" init-method="start">
    <property name="jobRunnerClass" value="com.github.ltsopensource.example.support.MyJobRunner"/>
    <property name="bizLoggerLevel" value="INFO"/>
    <property name="clusterName" value="test_cluster"/>
    <property name="registryAddress" value="zookeeper://127.0.0.1:2181"/>
    <property name="nodeGroup" value="test_trade_TaskTracker"/>
    <property name="workThreads" value="20"/>
    <property name="masterChangeListeners">
        <list>
            <bean class="com.github.ltsopensource.example.support.MasterChangeListenerImpl"/>
        </list>
    </property>
    <property name="configs">
        <props>
            <prop key="job.fail.store">leveldb</prop>
        </props>
    </property>
</bean>
```
### Spring annotation mode start
```java
@Configuration
public class LTSSpringConfig implements ApplicationContextAware {
    private ApplicationContext applicationContext;
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
	@Bean(name = "taskTracker")
    public TaskTracker getTaskTracker() throws Exception {
        TaskTrackerAnnotationFactoryBean factoryBean = new TaskTrackerAnnotationFactoryBean();
        factoryBean.setApplicationContext(applicationContext);
        factoryBean.setClusterName("test_cluster");
        factoryBean.setJobRunnerClass(MyJobRunner.class);
        factoryBean.setNodeGroup("test_trade_TaskTracker");
        factoryBean.setBizLoggerLevel("INFO");
        factoryBean.setRegistryAddress("zookeeper://127.0.0.1:2181");
        factoryBean.setMasterChangeListeners(new MasterChangeListener[]{
                new MasterChangeListenerImpl()
        });
        factoryBean.setWorkThreads(20);
        Properties configs = new Properties();
        configs.setProperty("job.fail.store", "leveldb");
        factoryBean.setConfigs(configs);

        factoryBean.afterPropertiesSet();
//        factoryBean.start();
        return factoryBean.getObject();
    }
}
```
## Parameter Description
[Parameter description](https://qq254963746.gitbooks.io/lts/content/use/config-name.html)

## Recommendations
Generally, only one JobClient instance is needed in a JVM. Do not create a new JobClient instance for each task. This will greatly waste resources, because a JobClient can submit multiple tasks. The same JVM generally tries to keep only one TaskTracker instance as much as possible, and more resources may be wasted. When encountering a TaskTracker to run multiple tasks, please refer to "One TaskTracker to perform multiple tasks" below.
## One TaskTracker performs multiple tasks
Sometimes, business scenarios need to perform multiple tasks, some people will ask, do you need a TaskTracker to perform each task type. My answer is no. If in a JVM, it is best to use one TaskTracker to run multiple tasks, because using multiple TaskTracker instances in a JVM is a waste of resources (of course, when you have a lot of tasks, you can Use a TaskTracker node to perform this task alone). So how can we implement a TaskTracker to perform multiple tasks? The following is a reference example I gave.

```java
/**
 * 总入口，在 taskTracker.setJobRunnerClass(JobRunnerDispatcher.class)
 * JobClient 提交 任务时指定 Job 类型  job.setParam("type", "aType")
 */
public class JobRunnerDispatcher implements JobRunner {

    private static final ConcurrentHashMap<String/*type*/, JobRunner>
            JOB_RUNNER_MAP = new ConcurrentHashMap<String, JobRunner>();

    static {
        JOB_RUNNER_MAP.put("aType", new JobRunnerA()); // 也可以从Spring中拿
        JOB_RUNNER_MAP.put("bType", new JobRunnerB());
    }

    @Override
    public Result run(JobContext jobContext) throws Throwable {
        Job job = jobContext.getJob();
        String type = job.getParam("type");
        return JOB_RUNNER_MAP.get(type).run(job);
    }
}

class JobRunnerA implements JobRunner {
    @Override
    public Result run(JobContext jobContext) throws Throwable {
        //  TODO A类型Job的逻辑
        return null;
    }
}

class JobRunnerB implements JobRunner {
    @Override
    public Result run(JobContext jobContext) throws Throwable {
        // TODO B类型Job的逻辑
        return null;
    }
}
```
## TaskTracker's JobRunner test
Generally, when writing TaskTracker, you only need to test whether the implementation logic of JobRunner is correct, and you don't want to start LTS for remote testing. In order to facilitate testing, LTS provides a quick test method of JobRunner. Your own test class can integrate `com.github.ltsopensource.tasktracker.runner.JobRunnerTester`, and implement the `initContext` and `newJobRunner` methods. Such as the example in [lts-examples](https://github.com/ltsopensource/lts-examples)：

```java
public class TestJobRunnerTester extends JobRunnerTester {

    public static void main(String[] args) throws Throwable {
        //  Mock Job 数据
        Job job = new Job();
        job.setTaskId("2313213");

        JobContext jobContext = new JobContext();
        jobContext.setJob(job);

        JobExtInfo jobExtInfo = new JobExtInfo();
        jobExtInfo.setRetry(false);

        jobContext.setJobExtInfo(jobExtInfo);

        // 运行测试
        TestJobRunnerTester tester = new TestJobRunnerTester();
        Result result = tester.run(jobContext);
        System.out.println(JSON.toJSONString(result));
    }

    @Override
    protected void initContext() {
        // TODO 初始化Spring容器
    }

    @Override
    protected JobRunner newJobRunner() {
        return new TestJobRunner();
    }
}
```

## Spring Quartz Cron task seamless access
For Quartz's Cron task, you only need to add a code in the Spring configuration to access the LTS platform.

```xml
<bean class="com.github.ltsopensource.spring.quartz.QuartzLTSProxyBean">
    <property name="clusterName" value="test_cluster"/>
    <property name="registryAddress" value="zookeeper://127.0.0.1:2181"/>
    <property name="nodeGroup" value="quartz_test_group"/>
</bean>
```
## Spring Boot 支持

```java
@SpringBootApplication
@EnableJobTracker       // 启动JobTracker
@EnableJobClient        // 启动JobClient
@EnableTaskTracker      // 启动TaskTracker
@EnableMonitor          // 启动Monitor
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

All that is left is to add the corresponding configuration in application.properties. For details, see the example in the `com.github.ltsopensource.examples.springboot` package in lts-example.

## Multiple network card selection problem
When the machine has two network cards in the internal network, sometimes, the user wants to let the traffic of LTS go to the external network card, then it is necessary to change the mapping address of the host name to the address of the external network card in the host, and the internal network is the same. .

## About node identification
If the node ID is set when the node is started, LTS will set a UUID as the node ID by default, which will be less readable, but it can guarantee the uniqueness of each node. If the user can guarantee the uniqueness of the node ID, you can pass `setIdentity` to set, for example, if each node is deployed on a machine (a virtual machine), then identity can be set to the host name

## SPI extension description
Support SPI extensions of JobLogger, JobQueue, etc.

## [Compare with other solutions](https://qq254963746.gitbooks.io/lts/content/introduce/compareother.html)


## LTS-Admin starts with jetty (default), and hangs up from time to time
See [issue#389](https://github.com/ltsopensource/light-task-scheduler/issues/389)


# hiring! ! !
Working years more than three years

Educational requirements

Expectation level P6 (Senior Java Engineer)/P7 (Technical Expert)

Job description  

The member platform is responsible for the user system of the Alibaba Group, supports the user needs of various business lines within the group, and supports the user pass and business pass of the group's external cooperation.
Including user login & authorization, session system, registration, account management, account security and other functions at each end, underlying user information services, session and credential management, etc., it is one of the core product lines of the group, carrying hundreds of billions of calls per day Volume, peak tens of millions of QPS, and global distributed hybrid cloud architecture, etc.

As a software engineer, you will work on our core products, which provide key functions for our business infrastructure,
Depending on your interests and experience, you can work in one or more of the following areas: globalization, user experience, data security, machine learning, system high availability, etc.

1. Independently complete the system analysis and design of small and medium-sized projects, and lead the completion of detailed design and coding tasks to ensure the progress and quality of the project;
2. Be able to complete the task of code review in the team, ensure the validity and correctness of the relevant code, and be able to provide relevant performance and stability suggestions through the code review;
3. Participate in the construction of a universal, flexible, and intelligent business support platform to support complex businesses with multiple scenarios at the upper level.
job requirements  
1. A solid Java programming foundation, familiar with commonly used Java open source frameworks;
2. Practical experience in developing high-performance and high-availability data applications based on databases, caches, and distributed storage, and proficient in the LINUX operating system;
3. Have a good ability to identify and design general frameworks and modules;
4. Love technology, work earnestly and rigorously, have a nearly demanding awareness of system quality, and be good at communication and teamwork;
5. Experience in the development and design of large-scale e-commerce websites or core systems in the financial industry is preferred;
6. Work experience in big data processing, algorithms, and machine learning is preferred.
If you are interested, you can send your resume to hugui.hg@alibaba-inc.com. Welcome to post
