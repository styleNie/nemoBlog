<!--
---
title: "crontab执行spark任务"      
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
# <center>crontab执行spark任务</center>    

crontab执行spark任务遇到的问题    
flow.sh
```bash
#!/usr/bin/env bash
source ~/.bash_profile

#export JAVA_HOME=/software/servers/jdk1.7.0_67
#export HADOOP_CONF_DIR=/hadoop_conf

/spark/bin/spark-submit \
--master yarn \
--deploy-mode cluster \
--driver-memory 10G \
--executor-memory 10G \
--verbose -v \
--conf spark.shuffle.service.enabled=true \
--conf spark.dynamicAllocation.enabled=true \
--conf spark.dynamicAllocation.minExecutors=1 \
--conf spark.dynamicAllocation.maxExecutors=200 \
--conf spark.sql.autoBroadcastJoinThreshold=200000000 \
--executor-cores 8 \
--conf spark.yarn.executor.memoryOverhead=10G \
--conf spark.broadcast.compress=true \
--conf spark.rdd.compress=true \
--conf spark.speculation=true \
--conf spark.driver.extraLibraryPath=/software/servers/hadoop-2.7.1/lib/native \
--class cronflow2.cron \
/jar-with-dependencies.jar
```



crontab使用了自己独立的一套环境变量，与当前linux用户的path是不一样的。因此很多依赖于PATH的命令语句都会无法执行(比如依赖于python，或者依赖于java)    

解决方法1：   
在当前用户的linux shell下执行下列命令，获取当前的path信息，复制到剪贴版中：   
`echo $PATH`
将PATH信息加入到crontab中：    
`crontab -e`    
在文件首部新增两个空行，在第一行增加下列信息。其中`<$PATH>`改为你剪贴板中的值：    
`PATH=<$PATH>`    

解决方法2：
在脚本开头添加 `source ~/.bash_profile`