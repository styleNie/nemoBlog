---
title: "Spark监听器"    
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

# <center>Spark监听器</center>
MySparkListener.scala   
```scala
import org.apache.spark.scheduler._
class MySparkListener extends SparkListener {
  override def onJobStart(jobStart: SparkListenerJobStart): Unit = {
    cron.jobInfo.put(jobStart.jobId,jobStart.time)   // jobid  与job开始时间
    println("Job Start: JobId->" + jobStart.jobId)
    println("Start time: " + jobStart.time+ "->"+ util.timeStampToString(jobStart.time))
  }

  override def onJobEnd(jobEnd: SparkListenerJobEnd) {
    println("Job End:" + jobEnd.jobResult.getClass + ",Id:" + jobEnd.jobId)
    import scala.collection.JavaConversions._
    for (key <- cron.jobInfo.keySet()) {
      if (cron.jobInfo.get(key)==jobEnd.jobId) {         //job执行完毕，将其从jobInfo中移除
        cron.jobInfo.remove(key)
      }
    }
  }

  override def onApplicationStart(applicationStart: SparkListenerApplicationStart): Unit = synchronized {
    val appId = applicationStart.appId.get
    println("appId===========" + appId)
    //记录app的Id，用于后续处理：
  }

  override def onApplicationEnd(applicationEnd: SparkListenerApplicationEnd) {
    println("app:end")
  }
}
```    

MySparkListener类继承自spark开放的抽象sparkListener类，监听sparkJob的执行信息，job开始执行时，将jobid和startTime添加到全局变量jobInfo中，job执行完毕，则从jonInfo中移除。    


注册监听器       
```scala 
val conf = new SparkConf()
val sc = new SparkContext(conf)
sc.addSparkListener(new MySparkListener)    // 添加监听器
```