---
title: "pySpark条件过滤"    
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

# <center>pySpark条件过滤</center>          

## 	应用场景     
hiveSQL中不方便处理的复杂逻辑，需要用户自己写函数处理，此处需要找出fdm_search_suggest_processed_exposure_sz表中的requestquery字段包含五个汉字笔划的记录    

## 第一种写法   
```python     
from pyspark import SparkContext
from pyspark import SparkConf
from pyspark.sql import HiveContext
from pyspark.sql.types import *
from pyspark.sql.functions import udf,col
conf = SparkConf()
conf.set('spark.sql.codegen', 'True')
sc = SparkContext(conf=conf)
hiveCtx = HiveContext(sc)

def testFun1(dt):
    bihua = [u'⼀', u'⼁', u'⼃', u'乛', u'⼂']

    def bihua_filter(bihua):
        def _bihua_filter(y):
            res = 0
            for i in range(5):
                if bihua[i] in y:
                    res += 1
            return res > 0

        return udf(_bihua_filter, BooleanType())
hiveCtx.sql("""
select dt,requestquery from mytable
where dt ='{0}'
""".format(dt)).repartition(160).where(bihua_filter(bihua)(col("requestquery"))).registerTempTable("tmp_show")
```       

## 第二种       
```python
def testFun2(dt):
    bihua = [u'⼀', u'⼁', u'⼃', u'乛', u'⼂']

    def bihua_filter(bihua):
        def _bihua_filter(y):
            res = 0
            for i in range(5):
                if bihua[i] in y:
                    res += 1
            return res > 0

        return udf(_bihua_filter, BooleanType())

    # compare_udf = udf(_bihua_filter, BooleanType())

    rowRDD=hiveCtx.sql("""
        select dt,requestquery from mytable
          where dt ='{0}'
        """.format(dt)).repartition(160).rdd
    rowRDD.filter(bihua_filter(bihua)(col('requestquery')))

    schema = StructType([StructField("dt", StringType(), True),
                    StructField("requestquery", StringType(), True)
                    ])
    tmp_show = hiveCtx.createDataFrame(rowRDD, schema)
tmp_show.registerTempTable("tmp_show")
```       

## 错误写法         
```python
def testFun3(dt):
    bihua = [u'⼀', u'⼁', u'⼃', u'乛', u'⼂']

    def bihua_filter(bihua,y):
        res = 0
        for i in range(5):
            if bihua[i] in y:
                res += 1
        return res > 0

    compare_udf = udf(bihua_filter, BooleanType())

    rowRDD=hiveCtx.sql("""
        select dt,requestquery from mytable
          where dt ='{0}'
        """.format(dt)).repartition(160).rdd
    rowRDD.filter(compare_udf(bihua,col('requestquery')))

    schema = StructType([StructField("dt", StringType(), True),
                    StructField("requestquery", StringType(), True)
                    ])
    tmp_show = hiveCtx.createDataFrame(rowRDD, schema)
tmp_show.registerTempTable("tmp_show")
```          

<img src=./pictures/pyspark1.png /> 