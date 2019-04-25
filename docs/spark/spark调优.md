---
title: "Spark调优"    
author:     
date: April 02, 2018    

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

# <center>Spark调优</center>
   

## 0 背景介绍   
&emsp;&emsp;近期遇到一个对时效性要求较高的hive表的处理，逻辑并不复杂，即从gdm表(以lzo格式存储的hive表)中读取数据，与app表(以orc格式存储的hive表)join得到最终的结果表sdl表。下面是最开始的代码(省去相关业务细节)：   
```python
# coding: utf-8
import sys
import os
reload(sys)
sys.setdefaultencoding("utf-8")

from datetime import datetime
from datetime import timedelta
import json

from pyspark import SparkConf
from pyspark import SparkContext
from pyspark import Row
from pyspark.sql import HiveContext
from pyspark.sql.types import *

def transform(iters):
    for fld in iters:
        version = fld.version
        plant = fld.plant
        id = fld.id
        param = fld.param
        uniq = fld.browser_uniq
        user = fld.user
        # 部分字段处理逻辑省去
        yield Row(ext=param,id=id,uniq=uniq,user=user,version=version)

def fun_main():
    schema = StructType([
            StructField("ext", MapType(StringType(), StringType()), True),
            StructField("id", StringType(), True),
            StructField("uniq", StringType(), True),
            StructField("user", StringType(), True),
            StructField("version", StringType(), True)

        ])
    sql = """
    select id, param,user,uniq,version, plant
    from gdm
    where dt='{dt_str}' and id in ('{id_list}')
        and (substring(version,1,3)>='5' and uniq is not null)
    """.format(dt_str=dt_str, app_event_id_list=id_list)
    print sql
    hiveCtx.createDataFrame(hiveCtx.sql(sql).repartition(3000).rdd.mapPartitions(lambda iters: transform(iters)),
                            schema).registerTempTable("tmptransform")
    sql="""
    select  nvl(b.key_word,'-') key_word,a.sk,a.index,b.is_active,b.wids_info,a.id,a.uniq, a.user
    from (
        select ext['sk'] as sk, ext['id'] as id, ext['index'] as index,uniq, user
        from tmptransform 
    ) a left join (
        select  is_active, ext_columns['wids_info'] as wids_info, key_word, id
        from app where dt = '{0}' 
    ) b on a.id =b.id
    """
    hiveCtx.sql(sql.format(dt)).repartition(50).registerTempTable('rel')
    sql="""
        insert overwrite table   sdl     partition(dt='{0}')
        select * from rel
    """
    hiveCtx.sql(sql.format(dt))

if __name__ == "__main__":
    # 传入标准10位日期2017-01-01
    dt = sys.argv[1]
    dt_str = datetime.strptime(dt, "%Y-%m-%d").date()
    dt_int = dt_str.strftime("%Y%m%d")
    dt_str_25 = (dt_str - timedelta(days=int(25))).strftime("%Y-%m-%d")

    conf = SparkConf()
    conf.set('spark.sql.codegen', 'True')
    conf.set('spark.rdd.compress', 'True')
    conf.set('spark.broadcast.compress', 'True')
    sc = SparkContext(conf=conf, appName='spark')
    sc.setLogLevel("WARN")
    hiveCtx = HiveContext(sc)
    hiveCtx.setConf("hive.exec.dynamic.partition.mode", "nonstrict")
    hiveCtx.setConf("hive.auto.convert.join", "true")
    hiveCtx.setConf("spark.sql.shuffle.partitions", "2000")
    hiveCtx.setConf("spark.default.parallelism", "2000")
    hiveCtx.setConf("spark.shuffle.consolidateFiles", "true")
    hiveCtx.setConf("spark.shuffle.compress", "true")
    fun_main()

    print "work done!"
```   

&emsp;&emsp;由于业务要求，结果表sdl必须在3点半之前跑出来，而gdm表一般在2点40到3点之间才能出来，上面这段代码快则30分钟，慢则1个半小时(受集群资源状况影响)，显然是无法令人接受的。下面是2018-03-28号的运行情况。    

<div align=center><img src=./pictures/spark1.png /> </div>  
<center>图1 </center> 

&emsp;&emsp;解决思路分两部分，一是查看集群资源使用情况，解决资源问题，资源不够，一切都是白费蜡；二是代码层面的优化，下面着重叙述代码调优的过程。   

## 1 降内存   
&emsp;&emsp;看到图1中只有一个job,分为了5个stage,分析后，stage0对应的是读取gdm表，stage2对应读取app表，stage4为sdl表写入。gdm表读取了3.8T的lzo压缩文件，最后的shuffle Write只有44.7G，所以首先从这里入手。   
&emsp;&emsp;查看gdm表的存储目录，一共有4842个lzo文件，以及对应的索引文件，每个lzo文件大概是460M。 hive读取的时候将所有的数据都读进内存再过滤，能否边读边过滤呢，spark提供了newAPIHadoopFile接口用于读取各种不同类型的文件，修改代码如下：   
```python
# coding: utf-8
import sys
import os
reload(sys)
sys.setdefaultencoding("utf-8")

from datetime import datetime
from datetime import timedelta
import json

from pyspark import SparkConf
from pyspark import SparkContext
from pyspark import Row
from pyspark.sql import HiveContext
from pyspark.sql.types import *

def check(iters):
    for item in iters:
        fld = item[1].split("\t")  # lzo文件用 \t 分隔字段
        id = fld[23]
        param = fld[22]
        user = fld[20]
        uniq = fld[4]
        version = fld[6]
        plant = fld[5]
        # 部分字段处理逻辑省去
        yield Row(ext=param,id=id,uniq=uniq,user=user,version=version)

def fun_main():
    schema = StructType([
            StructField("ext", MapType(StringType(), StringType()), True),
            StructField("id", StringType(), True),
            StructField("uniq", StringType(), True),
            StructField("user", StringType(), True),
            StructField("version", StringType(), True)

        ])
    filerdd = sc.newAPIHadoopFile(
        "hdfs://ns1/user/gdm/dt={dt_str}/*.lzo".format(dt_str=dt_str),
        "com.hadoop.mapreduce.LzoTextInputFormat", "org.apache.hadoop.io.LongWritable",
        "org.apache.hadoop.io.Text").mapPartitions(lambda iter: check(iter))

    hiveCtx.createDataFrame(filerdd, schema).registerTempTable("tmptransform")

    sql="""
    select  nvl(b.key_word,'-') key_word,a.sk,a.index,b.is_active,b.wids_info,a.id,a.uniq, a.user
    from (
        select ext['sk'] as sk, ext['id'] as id, ext['index'] as index,uniq, user
        from tmptransform 
    ) a left join (
        select  is_active, ext_columns['wids_info'] as wids_info, key_word, id
        from app where dt = '{0}' 
    ) b on a.id =b.id
    """
    hiveCtx.sql(sql.format(dt)).repartition(50).registerTempTable('rel')
    sql="""
        insert overwrite table   sdl     partition(dt='{0}')
        select * from rel
    """
    hiveCtx.sql(sql.format(dt))

if __name__ == "__main__":
    # 传入标准10位日期2017-01-01
    dt = sys.argv[1]
    dt_str = datetime.strptime(dt, "%Y-%m-%d").date()
    dt_int = dt_str.strftime("%Y%m%d")
    dt_str_25 = (dt_str - timedelta(days=int(25))).strftime("%Y-%m-%d")

    conf = SparkConf()
    conf.set('spark.sql.codegen', 'True')
    conf.set('spark.rdd.compress', 'True')
    conf.set('spark.broadcast.compress', 'True')
    sc = SparkContext(conf=conf, appName='spark')
    sc.setLogLevel("WARN")
    hiveCtx = HiveContext(sc)
    hiveCtx.setConf("hive.exec.dynamic.partition.mode", "nonstrict")
    hiveCtx.setConf("hive.auto.convert.join", "true")
    hiveCtx.setConf("spark.sql.shuffle.partitions", "2000")
    hiveCtx.setConf("spark.default.parallelism", "2000")
    hiveCtx.setConf("spark.shuffle.consolidateFiles", "true")
    hiveCtx.setConf("spark.shuffle.compress", "true")
    fun_main()

    print "work done!"
```    

<div align=center><img src=./pictures/spark2.png /> </div> 
<center>图2 </center>    

注意stage2,可以看到读入的数据量显著降低了。   

## 2 调整map数量   
&emsp;&emsp;图2中看到读入的数据量虽然降低了，但是注意到，目录下一共只有4842个lzo文件，但是stage2确有33104个task,且每个task大概是10s钟的持续时间，也就是说，每个lzo文件用了多个线程去读取，这样频繁的打开文件指针，也会带来较大的时间开销，理想情况下是每个文件对应一个map，处理1分钟左右，这次改用textFile接口读取数据，修改如下：   
```python
# coding: utf-8
import sys
import os
reload(sys)
sys.setdefaultencoding("utf-8")

from datetime import datetime
from datetime import timedelta
import json

from pyspark import SparkConf
from pyspark import SparkContext
from pyspark import Row
from pyspark.sql import HiveContext
from pyspark.sql.types import *

def check(iters):
    for item in iters:
        fld = item.split("\t")  # lzo文件用 \t 分隔字段
        id = fld[23]
        param = fld[22]
        user = fld[20]
        uniq = fld[4]
        version = fld[6]
        plant = fld[5]
        # 部分字段处理逻辑省去
        yield Row(ext=param,id=id,uniq=uniq,user=user,version=version)

def fun_main():
    schema = StructType([
            StructField("ext", MapType(StringType(), StringType()), True),
            StructField("id", StringType(), True),
            StructField("uniq", StringType(), True),
            StructField("user", StringType(), True),
            StructField("version", StringType(), True)

        ])
    filerdd = sc.textFile("hdfs://ns1/user/gdm/dt={dt_str}/*.lzo".format(dt_str=dt_str)).mapPartitions(lambda iter: check(iter))

    hiveCtx.createDataFrame(filerdd, schema).registerTempTable("tmptransform")

    sql="""
    select  nvl(b.key_word,'-') key_word,a.sk,a.index,b.is_active,b.wids_info,a.id,a.uniq, a.user
    from (
        select ext['sk'] as sk, ext['id'] as id, ext['index'] as index,uniq, user
        from tmptransform 
    ) a left join (
        select  is_active, ext_columns['wids_info'] as wids_info, key_word, id
        from app where dt = '{0}' 
    ) b on a.id =b.id
    """
    hiveCtx.sql(sql.format(dt)).repartition(50).registerTempTable('rel')
    sql="""
        insert overwrite table   sdl     partition(dt='{0}')
        select * from rel
    """
    hiveCtx.sql(sql.format(dt))

if __name__ == "__main__":
    # 传入标准10位日期2017-01-01
    dt = sys.argv[1]
    dt_str = datetime.strptime(dt, "%Y-%m-%d").date()
    dt_int = dt_str.strftime("%Y%m%d")
    dt_str_25 = (dt_str - timedelta(days=int(25))).strftime("%Y-%m-%d")

    conf = SparkConf()
    conf.set('spark.sql.codegen', 'True')
    conf.set('spark.rdd.compress', 'True')
    conf.set('spark.broadcast.compress', 'True')
    sc = SparkContext(conf=conf, appName='spark')
    sc.setLogLevel("WARN")
    hiveCtx = HiveContext(sc)
    hiveCtx.setConf("hive.exec.dynamic.partition.mode", "nonstrict")
    hiveCtx.setConf("hive.auto.convert.join", "true")
    hiveCtx.setConf("spark.sql.shuffle.partitions", "2000")
    hiveCtx.setConf("spark.default.parallelism", "2000")
    hiveCtx.setConf("spark.shuffle.consolidateFiles", "true")
    hiveCtx.setConf("spark.shuffle.compress", "true")
    fun_main()

    print "work done!"
```     

<div align=center><img src=./pictures/spark3.png /> </div> 
<center>图3 </center>    
注意到stage1读取gdm表下4842个文件用了4842个task,每个task持续2.5分钟左右。   

## 3 集群资源申请策略调整   
&emsp;&emsp;经过前两步的优化，现在任务所需的内存已经显著降下来了，所以任务变成了cpu密集型，这里调整原来的资源申请策略，一是**降低单个Executor所需的内存**，改为3G；二是根据集群采用的动态资源分配方式，**调整申请时的最小和最大Executor数量**,降低资源申请时等待的时间，具体的是   
spark.dynamicAllocation.minExecutors调小，   
spark.dynamicAllocation.maxExecutors调大。      

## 4 优化orc文件读取   
&emsp;&emsp;前面主要是对lzo的优化，到目前为止，没有更进一步的措施了，所以考虑优化orc文件读取。同样的app表路径下只有1500个文件，每个文件大概180M，理想情况下应该也是1500个map最佳；由于orc特殊的列式存储结构，已经做了相关优化，所以前面的办法不适用，这里从orc的读取策略入手。   
Spark读取orc文件默认采用HYBRID策略。   
```java
HIVE_ORC_SPLIT_STRATEGY("hive.exec.orc.split.strategy", "HYBRID", new StringSet(new String[]{"HYBRID", "BI", "ETL"}),      
 "This is not a user level config. BI strategy is used when the requirement is to spend less time in split generation as opposed       
to query execution (split generation does not read or cache file footers). ETL strategy is used when spending little more time in       
split generation is acceptable (split generation reads and caches file footers). HYBRID chooses between the above strategies based       
on heuristics."),
```     

&emsp;&emsp;HYBRID策略：Spark Driver启动的时候，会去nameNode读取元数据，根据文件总大小和文件个数计算一个文件的平均大小，如果这个平均值大于默认256M的时候就会触发ETL策略。ETL策略就会去DataNode上读取orc文件的head等信息，如果stripe个数多或元数据信息太大就会导致Driver 产生FUll GC，这个时候就会表现为Driver启动到Task执行间隔时间太久的现象。   

处理方案   
spark 1.6.2:   
```scala
val hiveContext = new HiveContext(sc)
// 默认64M，即代表在压缩前数据量累计到64M就会产生一个stripe。与之对应的hive.exec.orc.default.row.index.stride=10000可以控制有多少行是产生一个stripe。
// 调整这个参数可控制单个文件中stripe的个数，不配置单个文件stripe过多，影响下游使用，如果配置了ETL切分策略或启发式触发了ETL切分策略，就会使得Driver读取DataNode元数据太大，进而导致频繁GC，使得计算Partition的时间太长难以接受。
hiveContext.setConf("hive.exec.orc.default.stripe.size","268435456")
// 总共有三种策略{"HYBRID", "BI", "ETL"}), 默认是"HYBRID","This is not a user level config. BI strategy is used when the requirement is to spend less time in split generation as opposed to query execution (split generation does not read or cache file footers). ETL strategy is used when spending little more time in split generation is acceptable (split generation reads and caches file footers). HYBRID chooses between the above strategies based on heuristics."),
// 如果不配置，当orc文件大小大于spark框架估算的平均值256M时，会触发ETL策略，导致Driver读取DataNode数据切分split花费大量的时间。
hiveContext.setConf("hive.exec.orc.split.strategy", "BI")
```   

spark2.2.0:   
```scala
// 创建一个支持Hive的SparkSession
val sparkSession = SparkSession
  .builder()
  .appName("PvMvToBase")
  // 默认64M，即代表在压缩前数据量累计到64M就会产生一个stripe。与之对应的hive.exec.orc.default.row.index.stride=10000可以控制有多少行是产生一个stripe。
  // 调整这个参数可控制单个文件中stripe的个数，不配置单个文件stripe过多，影响下游使用，如果配置了ETL切分策略或启发式触发了ETL切分策略，就会使得Driver读取DataNode元数据太大，进而导致频繁GC，使得计算Partition的时间太长难以接受。
  .config("hive.exec.orc.default.stripe.size", 268435456L)
  // 总共有三种策略{"HYBRID", "BI", "ETL"}), 默认是"HYBRID","This is not a user level config. BI strategy is used when the requirement is to spend less time in split generation as opposed to query execution (split generation does not read or cache file footers). ETL strategy is used when spending little more time in split generation is acceptable (split generation reads and caches file footers). HYBRID chooses between the above strategies based on heuristics."),
  // 如果不配置，当orc文件大小大于spark框架估算的平均值256M时，会触发ETL策略，导致Driver读取DataNode数据切分split花费大量的时间。
  .config("hive.exec.orc.split.strategy", "BI")
  .enableHiveSupport()
  .getOrCreate()
```     

在代码中加入   
```python
hiveCtx.setConf("hive.exec.orc.default.stripe.size", "268435456")
hiveCtx.setConf("hive.exec.orc.split.strategy", "BI")
```   
加入上述策略后，app表的读取用了1500个map，达到目的。   




Reference:   
1.[spark开发笔记-scala 读lzo文件两种写法](https://blog.csdn.net/yanshu2012/article/details/54140038)   
2.[【总结】spark按文本格式和Lzo格式处理Lzo压缩文件的比较](http://blog.51cto.com/10120275/1954601)   
3.[spark SQl读取ORC文件](https://blog.csdn.net/aijiudu/article/details/78616064?locationNum=9&fps=1)
