---
layout: post
title: 定时任务调度Quartz实现
date: 2019-2-26 17:32:44
catalog: true
tags:
    - Quartz
---

## 定时任务应用场景

- 未支付订单的关闭
- 大数据系统生成报表
- 信用卡账单、还款通知
- ······

## 调度器的类型

- 系统调度器：Linux crontab
- Java：Timer、ScheduledExecutorService
- Spring Task
- Quartz
- ElasticJob

## 传统调度器的缺点

1、无法动态修改

2、无法集群部署

## Quartz

#### 简介

Quartz主要由三部分组成：

- Scheduler调度器
- Job任务
- Trigger触发器

#### 快速上手

1、添加pom依赖

```xml
<dependency>
    <groupId>org.quartz-scheduler</groupId>
    <artifactId>quartz</artifactId>
    <version>2.2.1</version>
</dependency>
<dependency>
    <groupId>org.quartz-scheduler</groupId>
    <artifactId>quartz-jobs</artifactId>
    <version>2.2.1</version>
</dependency>
```

2、创建Job任务

```java
public class MyJob implements Job {

    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        System.out.println(dateFormat.format(new Date()) + " hello quartz!");
    }
}
```

3、创建测试类

```java
public class MyScheduler {
    public static void main(String[] args) throws SchedulerException {
        // 获取调度器实例
        Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();
        // 启动调度器
        scheduler.start();
        // 定义任务
        JobDetail job = JobBuilder.newJob(MyJob.class)
                .withIdentity("fwj-job", "fwj-group")
                .build();
        // 定义触发器
        Trigger trigger = TriggerBuilder.newTrigger()
                .withIdentity("fwj-trigger", "fwj-group")
                .startNow()
                .withSchedule(SimpleScheduleBuilder.simpleSchedule().withIntervalInSeconds(5).repeatForever())
                .build();
        // 告诉调度器任务和使用的触发器
        scheduler.scheduleJob(job, trigger);
    }
}
```

#### 持久化到数据库

定时任务默认是持久化到内存，重启之后就不见了，这里使用MySQL存储调度器数据。实现任务列表查询、添加任务、删除任务、启动任务、暂停任务功能。

1、下载建表SQL语句

下载地址：http://www.quartz-scheduler.org/downloads/

或者复制下面的SQL：

```
#
# In your Quartz properties file, you'll need to set 
# org.quartz.jobStore.driverDelegateClass = org.quartz.impl.jdbcjobstore.StdJDBCDelegate
#
#
# By: Ron Cordell - roncordell
#  I didn't see this anywhere, so I thought I'd post it here. This is the script from Quartz to create the tables in a MySQL database, modified to use INNODB instead of MYISAM.

DROP TABLE IF EXISTS QRTZ_FIRED_TRIGGERS;
DROP TABLE IF EXISTS QRTZ_PAUSED_TRIGGER_GRPS;
DROP TABLE IF EXISTS QRTZ_SCHEDULER_STATE;
DROP TABLE IF EXISTS QRTZ_LOCKS;
DROP TABLE IF EXISTS QRTZ_SIMPLE_TRIGGERS;
DROP TABLE IF EXISTS QRTZ_SIMPROP_TRIGGERS;
DROP TABLE IF EXISTS QRTZ_CRON_TRIGGERS;
DROP TABLE IF EXISTS QRTZ_BLOB_TRIGGERS;
DROP TABLE IF EXISTS QRTZ_TRIGGERS;
DROP TABLE IF EXISTS QRTZ_JOB_DETAILS;
DROP TABLE IF EXISTS QRTZ_CALENDARS;

CREATE TABLE QRTZ_JOB_DETAILS(
SCHED_NAME VARCHAR(120) NOT NULL,
JOB_NAME VARCHAR(200) NOT NULL,
JOB_GROUP VARCHAR(200) NOT NULL,
DESCRIPTION VARCHAR(250) NULL,
JOB_CLASS_NAME VARCHAR(250) NOT NULL,
IS_DURABLE VARCHAR(1) NOT NULL,
IS_NONCONCURRENT VARCHAR(1) NOT NULL,
IS_UPDATE_DATA VARCHAR(1) NOT NULL,
REQUESTS_RECOVERY VARCHAR(1) NOT NULL,
JOB_DATA BLOB NULL,
PRIMARY KEY (SCHED_NAME,JOB_NAME,JOB_GROUP))
ENGINE=InnoDB;

CREATE TABLE QRTZ_TRIGGERS (
SCHED_NAME VARCHAR(120) NOT NULL,
TRIGGER_NAME VARCHAR(200) NOT NULL,
TRIGGER_GROUP VARCHAR(200) NOT NULL,
JOB_NAME VARCHAR(200) NOT NULL,
JOB_GROUP VARCHAR(200) NOT NULL,
DESCRIPTION VARCHAR(250) NULL,
NEXT_FIRE_TIME BIGINT(13) NULL,
PREV_FIRE_TIME BIGINT(13) NULL,
PRIORITY INTEGER NULL,
TRIGGER_STATE VARCHAR(16) NOT NULL,
TRIGGER_TYPE VARCHAR(8) NOT NULL,
START_TIME BIGINT(13) NOT NULL,
END_TIME BIGINT(13) NULL,
CALENDAR_NAME VARCHAR(200) NULL,
MISFIRE_INSTR SMALLINT(2) NULL,
JOB_DATA BLOB NULL,
PRIMARY KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP),
FOREIGN KEY (SCHED_NAME,JOB_NAME,JOB_GROUP)
REFERENCES QRTZ_JOB_DETAILS(SCHED_NAME,JOB_NAME,JOB_GROUP))
ENGINE=InnoDB;

CREATE TABLE QRTZ_SIMPLE_TRIGGERS (
SCHED_NAME VARCHAR(120) NOT NULL,
TRIGGER_NAME VARCHAR(200) NOT NULL,
TRIGGER_GROUP VARCHAR(200) NOT NULL,
REPEAT_COUNT BIGINT(7) NOT NULL,
REPEAT_INTERVAL BIGINT(12) NOT NULL,
TIMES_TRIGGERED BIGINT(10) NOT NULL,
PRIMARY KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP),
FOREIGN KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP)
REFERENCES QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP))
ENGINE=InnoDB;

CREATE TABLE QRTZ_CRON_TRIGGERS (
SCHED_NAME VARCHAR(120) NOT NULL,
TRIGGER_NAME VARCHAR(200) NOT NULL,
TRIGGER_GROUP VARCHAR(200) NOT NULL,
CRON_EXPRESSION VARCHAR(120) NOT NULL,
TIME_ZONE_ID VARCHAR(80),
PRIMARY KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP),
FOREIGN KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP)
REFERENCES QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP))
ENGINE=InnoDB;

CREATE TABLE QRTZ_SIMPROP_TRIGGERS
  (          
    SCHED_NAME VARCHAR(120) NOT NULL,
    TRIGGER_NAME VARCHAR(200) NOT NULL,
    TRIGGER_GROUP VARCHAR(200) NOT NULL,
    STR_PROP_1 VARCHAR(512) NULL,
    STR_PROP_2 VARCHAR(512) NULL,
    STR_PROP_3 VARCHAR(512) NULL,
    INT_PROP_1 INT NULL,
    INT_PROP_2 INT NULL,
    LONG_PROP_1 BIGINT NULL,
    LONG_PROP_2 BIGINT NULL,
    DEC_PROP_1 NUMERIC(13,4) NULL,
    DEC_PROP_2 NUMERIC(13,4) NULL,
    BOOL_PROP_1 VARCHAR(1) NULL,
    BOOL_PROP_2 VARCHAR(1) NULL,
    PRIMARY KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP),
    FOREIGN KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP) 
    REFERENCES QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP))
ENGINE=InnoDB;

CREATE TABLE QRTZ_BLOB_TRIGGERS (
SCHED_NAME VARCHAR(120) NOT NULL,
TRIGGER_NAME VARCHAR(200) NOT NULL,
TRIGGER_GROUP VARCHAR(200) NOT NULL,
BLOB_DATA BLOB NULL,
PRIMARY KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP),
INDEX (SCHED_NAME,TRIGGER_NAME, TRIGGER_GROUP),
FOREIGN KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP)
REFERENCES QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP))
ENGINE=InnoDB;

CREATE TABLE QRTZ_CALENDARS (
SCHED_NAME VARCHAR(120) NOT NULL,
CALENDAR_NAME VARCHAR(200) NOT NULL,
CALENDAR BLOB NOT NULL,
PRIMARY KEY (SCHED_NAME,CALENDAR_NAME))
ENGINE=InnoDB;

CREATE TABLE QRTZ_PAUSED_TRIGGER_GRPS (
SCHED_NAME VARCHAR(120) NOT NULL,
TRIGGER_GROUP VARCHAR(200) NOT NULL,
PRIMARY KEY (SCHED_NAME,TRIGGER_GROUP))
ENGINE=InnoDB;

CREATE TABLE QRTZ_FIRED_TRIGGERS (
SCHED_NAME VARCHAR(120) NOT NULL,
ENTRY_ID VARCHAR(95) NOT NULL,
TRIGGER_NAME VARCHAR(200) NOT NULL,
TRIGGER_GROUP VARCHAR(200) NOT NULL,
INSTANCE_NAME VARCHAR(200) NOT NULL,
FIRED_TIME BIGINT(13) NOT NULL,
SCHED_TIME BIGINT(13) NOT NULL,
PRIORITY INTEGER NOT NULL,
STATE VARCHAR(16) NOT NULL,
JOB_NAME VARCHAR(200) NULL,
JOB_GROUP VARCHAR(200) NULL,
IS_NONCONCURRENT VARCHAR(1) NULL,
REQUESTS_RECOVERY VARCHAR(1) NULL,
PRIMARY KEY (SCHED_NAME,ENTRY_ID))
ENGINE=InnoDB;

CREATE TABLE QRTZ_SCHEDULER_STATE (
SCHED_NAME VARCHAR(120) NOT NULL,
INSTANCE_NAME VARCHAR(200) NOT NULL,
LAST_CHECKIN_TIME BIGINT(13) NOT NULL,
CHECKIN_INTERVAL BIGINT(13) NOT NULL,
PRIMARY KEY (SCHED_NAME,INSTANCE_NAME))
ENGINE=InnoDB;

CREATE TABLE QRTZ_LOCKS (
SCHED_NAME VARCHAR(120) NOT NULL,
LOCK_NAME VARCHAR(40) NOT NULL,
PRIMARY KEY (SCHED_NAME,LOCK_NAME))
ENGINE=InnoDB;

CREATE INDEX IDX_QRTZ_J_REQ_RECOVERY ON QRTZ_JOB_DETAILS(SCHED_NAME,REQUESTS_RECOVERY);
CREATE INDEX IDX_QRTZ_J_GRP ON QRTZ_JOB_DETAILS(SCHED_NAME,JOB_GROUP);

CREATE INDEX IDX_QRTZ_T_J ON QRTZ_TRIGGERS(SCHED_NAME,JOB_NAME,JOB_GROUP);
CREATE INDEX IDX_QRTZ_T_JG ON QRTZ_TRIGGERS(SCHED_NAME,JOB_GROUP);
CREATE INDEX IDX_QRTZ_T_C ON QRTZ_TRIGGERS(SCHED_NAME,CALENDAR_NAME);
CREATE INDEX IDX_QRTZ_T_G ON QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_GROUP);
CREATE INDEX IDX_QRTZ_T_STATE ON QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_STATE);
CREATE INDEX IDX_QRTZ_T_N_STATE ON QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP,TRIGGER_STATE);
CREATE INDEX IDX_QRTZ_T_N_G_STATE ON QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_GROUP,TRIGGER_STATE);
CREATE INDEX IDX_QRTZ_T_NEXT_FIRE_TIME ON QRTZ_TRIGGERS(SCHED_NAME,NEXT_FIRE_TIME);
CREATE INDEX IDX_QRTZ_T_NFT_ST ON QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_STATE,NEXT_FIRE_TIME);
CREATE INDEX IDX_QRTZ_T_NFT_MISFIRE ON QRTZ_TRIGGERS(SCHED_NAME,MISFIRE_INSTR,NEXT_FIRE_TIME);
CREATE INDEX IDX_QRTZ_T_NFT_ST_MISFIRE ON QRTZ_TRIGGERS(SCHED_NAME,MISFIRE_INSTR,NEXT_FIRE_TIME,TRIGGER_STATE);
CREATE INDEX IDX_QRTZ_T_NFT_ST_MISFIRE_GRP ON QRTZ_TRIGGERS(SCHED_NAME,MISFIRE_INSTR,NEXT_FIRE_TIME,TRIGGER_GROUP,TRIGGER_STATE);

CREATE INDEX IDX_QRTZ_FT_TRIG_INST_NAME ON QRTZ_FIRED_TRIGGERS(SCHED_NAME,INSTANCE_NAME);
CREATE INDEX IDX_QRTZ_FT_INST_JOB_REQ_RCVRY ON QRTZ_FIRED_TRIGGERS(SCHED_NAME,INSTANCE_NAME,REQUESTS_RECOVERY);
CREATE INDEX IDX_QRTZ_FT_J_G ON QRTZ_FIRED_TRIGGERS(SCHED_NAME,JOB_NAME,JOB_GROUP);
CREATE INDEX IDX_QRTZ_FT_JG ON QRTZ_FIRED_TRIGGERS(SCHED_NAME,JOB_GROUP);
CREATE INDEX IDX_QRTZ_FT_T_G ON QRTZ_FIRED_TRIGGERS(SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP);
CREATE INDEX IDX_QRTZ_FT_TG ON QRTZ_FIRED_TRIGGERS(SCHED_NAME,TRIGGER_GROUP);

commit; 
```

2、添加`quartz.properties`配置文件

```properties
# 线程池配置
org.quartz.threadPool.class: org.quartz.simpl.SimpleThreadPool
org.quartz.threadPool.threadCount: 2
org.quartz.threadPool.threadPriority: 5

# JobStore 任务持久化配置
org.quartz.jobStore.class: org.quartz.impl.jdbcjobstore.JobStoreTX
org.quartz.jobStore.driverDelegateClass: org.quartz.impl.jdbcjobstore.StdJDBCDelegate

# 数据库别名
org.quartz.jobStore.dataSource: qzDS
org.quartz.jobStore.tablePrefix: QRTZ_

# 数据源
org.quartz.dataSource.qzDS.driver: com.mysql.jdbc.Driver
org.quartz.dataSource.qzDS.URL: jdbc:mysql://192.168.241.131:3306/quartz
org.quartz.dataSource.qzDS.user: root
org.quartz.dataSource.qzDS.password: 123456
org.quartz.dataSource.qzDS.maxConnection: 10
```

3、添加pom数据库依赖

```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
```

4、编写调度器工具类

```java
public class QuartzUtil {
    private static final Logger logger = LoggerFactory.getLogger(QuartzUtil.class);

    public static List<JobEntity> getList(String jobGroup) throws SchedulerException {
        Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();
        Set<JobKey> jobKeys = scheduler.getJobKeys(GroupMatcher.jobGroupEquals(jobGroup));
        List<JobEntity> jobEntityList = new ArrayList<>();
        for (JobKey jobKey : jobKeys) {
            JobEntity entity = new JobEntity();
            entity.setJobName(jobKey.getName());
            entity.setJobGroup(jobKey.getGroup());
            entity.setClassName(scheduler.getJobDetail(jobKey).getJobClass().getName());
            TriggerKey triggerKey = TriggerKey.triggerKey(jobKey.getName(), jobGroup);
            entity.setCron(((CronTrigger)scheduler.getTrigger(triggerKey)).getCronExpression());
            jobEntityList.add(entity);
        }
        return jobEntityList;
    }

    /**
     * 添加任务
     * @param className 类路径
     * @param jobName 任务名称
     * @param jobGroup 组别
     * @param cronExpression cron表达式
     * @throws Exception
     */
    public static void addJob(String className, String jobName, String jobGroup, String cronExpression)
            throws Exception {
        JobDetail jobDetail = JobBuilder.newJob(getClass(className).getClass())
                .withIdentity(jobName, jobGroup).build();
        CronScheduleBuilder scheduleBuilder = CronScheduleBuilder.cronSchedule(cronExpression)
                .withMisfireHandlingInstructionDoNothing();
        CronTrigger trigger = TriggerBuilder.newTrigger().withIdentity(jobName, jobGroup)
                .withSchedule(scheduleBuilder).build();
        try {
            Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();
            scheduler.scheduleJob(jobDetail, trigger);
            scheduler.pauseJob(JobKey.jobKey(jobName, jobGroup));
            logger.info("创建定时任务{}成功", jobName);
        } catch (SchedulerException e) {
            logger.error("创建定时任务失败：" + e);
            throw new SchedulerException("创建定时任务失败");
        }
    }

    /**
     * 暂停任务
     * @param jobName 任务名
     * @param jobGroup 组别
     * @throws SchedulerException
     */
    public static void pauseJob(String jobName, String jobGroup) throws SchedulerException {
        Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();
        logger.info("停用任务{}", jobName);
        scheduler.pauseJob(JobKey.jobKey(jobName, jobGroup));
    }

    /**
     * 启用一个定时任务
     * @param jobName 任务名称
     * @param jobGroup 组别
     * @throws SchedulerException
     */
    public static void resumeJob(String jobName, String jobGroup) throws SchedulerException {
        Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();
        logger.info("启用任务{}", jobName);
        scheduler.resumeJob(JobKey.jobKey(jobName, jobGroup));
        if (!scheduler.isStarted()) {
            scheduler.start();
        }
    }

    /**
     * 删除定时任务
     * @param jobName 任务名称
     * @param jobGroup 组别
     * @throws SchedulerException
     */
    public static void deleteJob(String jobName, String jobGroup) throws SchedulerException {
        Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();
        logger.info("删除任务{}", jobName);
        scheduler.pauseTrigger(TriggerKey.triggerKey(jobName, jobGroup));
        scheduler.unscheduleJob(TriggerKey.triggerKey(jobName, jobGroup));
        scheduler.deleteJob(JobKey.jobKey(jobName, jobGroup));
    }

    public static void updateScheduler(String jobName, String jobGroup, String cronExpression) throws SchedulerException {
        try {
            Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();
            TriggerKey triggerKey = TriggerKey.triggerKey(jobName, jobGroup);
            CronScheduleBuilder scheduleBuilder = CronScheduleBuilder.cronSchedule(cronExpression);
            CronTrigger trigger = (CronTrigger) scheduler.getTrigger(triggerKey);
            trigger = trigger.getTriggerBuilder().withIdentity(triggerKey).withSchedule(scheduleBuilder)
                    .startNow().build();
            scheduler.rescheduleJob(triggerKey, trigger);
            logger.info("更新定时任务{}成功", jobName);
        } catch (SchedulerException e) {
            logger.error("更新定时任务失败：" + e);
            throw new SchedulerException("更新定时任务失败");
        }
    }

    private static Job getClass(String className) throws ClassNotFoundException, IllegalAccessException, InstantiationException {
        Class<?> aClass = Class.forName(className);
        return (Job) aClass.newInstance();
    }
}
```

**注意：** 添加任务时，不启动调度器，先暂停任务，实现手动启动任务。

5、编写Controller接口

```java
@RestController
@RequestMapping("/job")
public class SysJobController {

    @GetMapping("/add")
    public String addJob(@RequestParam("name") String jobName,
                         @RequestParam("class") String className,
                         @RequestParam("cron") String cron) {
        try {
            QuartzUtil.addJob("com.fang.quartzdemo.ram." + className, jobName,
                    "fwj-group", cron);
            return "success";
        } catch (Exception e) {
            e.printStackTrace();
        }
        return "fail";
    }

    @GetMapping("/list")
    public List<JobEntity> getList(@RequestParam("group") String jobGroup) throws SchedulerException {
        return QuartzUtil.getList(jobGroup);
    }

    @GetMapping("/start")
    public String startJob(@RequestParam("name") String jobName, @RequestParam("group") String jobGroup) {
        try {
            QuartzUtil.resumeJob(jobName, jobGroup);
            return "success";
        } catch (SchedulerException e) {
            e.printStackTrace();
        }
        return "fail";
    }

    @GetMapping("/stop")
    public String stopJob(@RequestParam("name") String jobName, @RequestParam("group") String jobGroup) {
        try {
            QuartzUtil.pauseJob(jobName, jobGroup);
            return "success";
        } catch (SchedulerException e) {
            e.printStackTrace();
        }
        return "fail";
    }

    @GetMapping("/delete")
    public String deleteJob(@RequestParam("name") String jobName, @RequestParam("group") String jobGroup) {
        try {
            QuartzUtil.deleteJob(jobName, jobGroup);
            return "success";
        } catch (SchedulerException e) {
            e.printStackTrace();
        }
        return "fail";
    }
}
```

6、让调度器随Spring一起启动，这样程序启动后那些本来正常运行的定时器会继续运行。

```java
@Component
public class InitStartScheduler implements CommandLineRunner {
    private static final Logger logger = LoggerFactory.getLogger(InitStartScheduler.class);
    @Override
    public void run(String... args) throws Exception {
        logger.info("启动调度器...");
        Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();
        scheduler.start();
    }
}
```

- 需要添加`@Component`注解，不然不会随Spring启动。

## 参考

[Quartz官网](http://www.quartz-scheduler.org/)