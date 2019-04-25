<!--
---
title: "scala 定时任务执行"          
author:           
date: April 16, 2018          

toc:    
  depth_from: 1    
  depth_to: 6    
  ordered: false    
  ignoreLink:false    
  
html:    
  embed_local_images: true    
  embed_svg: true    
  offline: false    
  toc: false    

print_background: false    

export_on_save:    
  html: true    

---
-->
# <center>scala 定时任务执行</center>


quartz 版本
```html
<dependency>
<groupId>org.quartz-scheduler</groupId>
<artifactId>quartz</artifactId>
<version>2.2.3</version>
<!--<scope>provided</scope>-->
</dependency>
```    

java 实例1   
```java
import org.quartz.*;
import org.quartz.impl.StdSchedulerFactory;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;

public class myquartz implements Job {

    /**
     * Quartz requires a public empty constructor so that the
     * scheduler can instantiate the class whenever it needs.
     */
    public myquartz() {
    }

    /**
     * 该方法实现需要执行的任务
     */
    @SuppressWarnings("unchecked")
    //@Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
       System.out.println("Work Done!");
    }

    public static void main(String[] args) throws SchedulerException, InterruptedException {
        // 通过 schedulerFactory 获取一个调度器
        SchedulerFactory sf = new StdSchedulerFactory();
        Scheduler sched = sf.getScheduler();
        // 创建 jobDetail 实例，绑定 Job 实现类
        // 指明 job 的名称，所在组的名称，以及绑定 job 类
        JobDetail job = JobBuilder.newJob(myquartz.class).withIdentity("job1", "group1").build();
        // 定义调度触发规则
        // corn 表达式，先立即执行一次，然后每隔 5 秒执行 1 次
        Trigger trigger = TriggerBuilder.newTrigger()
                .withIdentity("trigger1", "group1")
                .withSchedule(CronScheduleBuilder.cronSchedule("*/5 * * * * ?"))
                .build();
        // 把作业和触发器注册到任务调度中
        sched.scheduleJob(job, trigger);
        // 启动计划程序（实际上直到调度器已经启动才会开始运行）
        sched.start();
        // 等待 10 秒，使我们的 job 有机会执行
       Thread.sleep(10000);
        // 等待作业执行完成时才关闭调度器
        sched.shutdown(true);
    }
}
```

java 实例2   
```java
import org.quartz.*;
import org.quartz.impl.StdSchedulerFactory;

import java.util.ArrayList;
import java.util.Date;
import java.util.List;

public class QuartzTest implements Job {
    /**
     * Quartz requires a public empty constructor so that the
     * scheduler can instantiate the class whenever it needs.
     */
    public QuartzTest() {
    }
    /**
     * 该方法实现需要执行的任务
     */
    @SuppressWarnings("unchecked")
    //@Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        // 从 context 中获取 instName, groupName 以及 dataMap
        String instName = context.getJobDetail().getKey().getName();
        String groupName = context.getJobDetail().getKey().getGroup();
        JobDataMap dataMap = context.getJobDetail().getJobDataMap();
        // 从 dataMap 中获取 myDescription, myValue 以及 myArray
        String myDescription = dataMap.getString("myDescription");
        int myValue = dataMap.getInt("myValue");
        List<String> myArray = (List<String>) dataMap.get("myArray");
        System.out.println("---> Instance = " + instName + ", group = " + groupName
                + ", description = " + myDescription + ", value =" + myValue
                + ", array item[0] = " + myArray.get(0));
        System.out.println("Runtime: " + new Date().toString() + " <---");
    }
    public static void main(String[] args) throws SchedulerException, InterruptedException {
        // 通过 schedulerFactory 获取一个调度器
        SchedulerFactory sf = new StdSchedulerFactory();
        Scheduler sched = sf.getScheduler();
        // 创建 jobDetail 实例，绑定 Job 实现类
        // 指明 job 的名称，所在组的名称，以及绑定 job 类
        JobDetail job = JobBuilder.newJob(QuartzTest.class).withIdentity("job1", "group1").build();
        // 定义调度触发规则
        // corn 表达式，先立即执行一次，然后每隔 5 秒执行 1 次
        Trigger trigger = TriggerBuilder.newTrigger()
                .withIdentity("trigger1", "group1")
                .withSchedule(CronScheduleBuilder.cronSchedule("*/5 * * * * ?"))
                .build();
        // 初始化参数传递到 job
        job.getJobDataMap().put("myDescription", "Hello Quartz");
        job.getJobDataMap().put("myValue", 1990);
        List<String> list = new ArrayList<String>();
        list.add("firstItem");
        job.getJobDataMap().put("myArray", list);
        // 把作业和触发器注册到任务调度中
        sched.scheduleJob(job, trigger);
        // 启动计划程序（实际上直到调度器已经启动才会开始运行）
        sched.start();
        // 等待 10 秒，使我们的 job 有机会执行
        Thread.sleep(10000);
        // 等待作业执行完成时才关闭调度器
        sched.shutdown(true);
    }
}
```   


java 实例3    
myJob.class   
```java
import java.text.SimpleDateFormat;
import java.util.Date;
import org.quartz.Job;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;
public class myJob implements Job {
    //@Override
    public void execute(JobExecutionContext arg0) throws JobExecutionException {
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss SSS");
        System.out.println(sdf.format(new Date()));
    }
}
```   

Test2.class    
```java
import static org.quartz.CronScheduleBuilder.cronSchedule;
import static org.quartz.JobBuilder.newJob;
import static org.quartz.TriggerBuilder.newTrigger;
import java.text.SimpleDateFormat;
import java.util.Date;
import org.quartz.CronTrigger;
import org.quartz.JobDetail;
import org.quartz.Scheduler;
import org.quartz.SchedulerFactory;
import org.quartz.impl.StdSchedulerFactory;

public class Test2 {
    public void go() throws Exception {
        // 首先，必需要取得一个Scheduler的引用
        SchedulerFactory sf = new StdSchedulerFactory();
        Scheduler sched = sf.getScheduler();
        //jobs可以在scheduled的sched.start()方法前被调用

        //job 1将每隔20秒执行一次
        JobDetail job = newJob(myJob.class).withIdentity("job1", "group1").build();
        CronTrigger trigger = newTrigger().withIdentity("trigger1", "group1")
                .withSchedule(cronSchedule("0 0/1 8-17 * * ?")).build();
        Date ft = sched.scheduleJob(job, trigger);
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss SSS");
        System.out.println(job.getKey() + " 已被安排执行于: " + sdf.format(ft) + "，并且以如下重复规则重复执行: " + trigger.getCronExpression());

        // job 2将每2分钟执行一次（在该分钟的第15秒)
        job = newJob(myJob.class).withIdentity("job2", "group1").build();
        trigger = newTrigger().withIdentity("trigger2", "group1").withSchedule(cronSchedule("15 0/2 * * * ?")).build();
        ft = sched.scheduleJob(job, trigger);
        System.out.println(job.getKey() + " 已被安排执行于: " + sdf.format(ft) + "，并且以如下重复规则重复执行: "+ trigger.getCronExpression());

        // 开始执行，start()方法被调用后，计时器就开始工作，计时调度中允许放入N个Job
        sched.start();
        try {
            //主线程等待一分钟
            Thread.sleep(3*60L * 1000L);
        } catch (Exception e) {}
        //关闭定时调度，定时器不再工作
        sched.shutdown(true);
    }

    public static void main(String[] args) throws Exception {
        Test2 test = new Test2();
        test.go();
    }
}
```     


scala 实例    
myjob.scala    
```scala
import java.util.{Calendar, Date, Properties}
import org.apache.spark.{SparkConf, SparkContext}
import org.apache.spark.sql.hive.HiveContext
import org.apache.spark.sql.{Row, SQLContext, SaveMode}
import org.apache.spark.sql.types.{StringType, StructField, StructType}
import org.quartz.Job
import org.quartz.JobExecutionContext
import org.quartz.JobExecutionException
class myjob() extends Job{
  println("myJob  ****************************")
  @throws[JobExecutionException]
  override def execute(arg0: JobExecutionContext): Unit = {
    val a= Main.sc.makeRDD(Array(1,2,3,3,4,5))
    a.foreach(x=>println(x))
    println("*******************")
  }
}
```   

Main.scala    
```scala
import crontest.myjob
import org.apache.spark.{SparkConf, SparkContext}
import org.apache.spark.sql.hive.HiveContext
import org.quartz.CronScheduleBuilder.cronSchedule
import org.quartz.JobBuilder.newJob
import org.quartz.TriggerBuilder.newTrigger
import org.quartz.impl.StdSchedulerFactory

object Main {
  val conf = new SparkConf().setAppName("flowBulkQuery").setMaster("local[4]")
  conf.set("spark.sql.codegen", "true")
  conf.set("spark.sql.inMemoryColumnarStorage.compressed", "true")
  conf.set("spark.sql.autoBroadcastJoinThreshold", "200000")
  conf.set("spark.sql.shuffle.partitions", "5")
  conf.set("spark.default.parallelism", "5")
  conf.set("spark.shuffle.consolidateFiles", "true")
  conf.set("mapreduce.job.reduces", "1000")
  conf.set("spark.shuffle.compress", "true")
  conf.set("spark.driver.maxResultSize", "1g")
  val sc = new SparkContext(conf)
  //sc.setLogLevel("WARN")
  val sqlContext = new HiveContext(sc)

  @throws[Exception]
  def go(): Unit = { // 首先，必需要取得一个Scheduler的引用
    val sf = new StdSchedulerFactory
    val sched = sf.getScheduler
    //jobs可以在scheduled的sched.start()方法前被调用
    //job 1将每隔20秒执行一次
    var job = newJob(classOf[myjob]).withIdentity("job1", "group1").build()
    var trigger = newTrigger.withIdentity("trigger1", "group1").withSchedule(cronSchedule("0/30 * * * * ?")).build
    var ft = sched.scheduleJob(job, trigger)
    job = newJob(classOf[myjob]).withIdentity("job2", "group1").build
    trigger = newTrigger.withIdentity("trigger2", "group1").withSchedule(cronSchedule("15 0/2 * * * ?")).build
    ft = sched.scheduleJob(job, trigger)
    sched.start()
    println("start  ****************************")
    try { //主线程等待一分钟
      Thread.sleep(5 * 60L * 1000L)
      println("try  ****************************")
    }
    catch {
      case e: Exception =>
    }
    //关闭定时调度，定时器不再工作
    sched.shutdown(true)
  }

  @throws[Exception]
  def main(args: Array[String]): Unit = {
    Main.go()
  }
}
```

refer:    
1. https://www.cnblogs.com/monian/p/3822980.html#    
2. http://blog.csdn.net/yuan8080/article/details/6583603     
3. http://blog.csdn.net/zixiao217/article/details/53075009    
4. https://segmentfault.com/a/1190000009128277    
