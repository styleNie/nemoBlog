---
title: "Spark 上运行 XGBoost 示例"    
author:     
date: March 02, 2018    

toc:    
  depth_from: 1    
  depth_to: 6    
  ordered: false    
---

# <center>Spark 上运行 XGBoost 示例</center>  

##  环境准备 
```html
linux
jdk 1.7+   
Spark 2.0+      
xgboost4j-spark-0.7-jar-with-dependencies.jar 
``` 
xgboost4j-spark 需要自己编译，不同环境下需要分别编译，可参考 [windows下在Java中使用xgboost 详细配置教程](http://blog.csdn.net/eddy_zheng/article/details/51049435)  和[XGBoost 官网说明](http://xgboost.readthedocs.io/en/latest/jvm/index.html),编译碰到的问题较多，没有成功，这里采用的是  [旭旭_哥](http://blog.csdn.net/luoyexuge/article/details/71422270) 提供的jar包   


## 代码实例
pom文件配置如下   
```html
<dependencies>
        <dependency>
            <groupId>ml.dmlc</groupId>
            <artifactId>xgboost4j-spark</artifactId>
            <version>0.7</version>
            <scope>provided</scope>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.apache.spark/spark-core_2.11 -->
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_2.11</artifactId>
            <version>${spark.version}</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-mllib_2.11</artifactId>
            <version>${spark.version}</version>
            <scope>provided</scope>
        </dependency>
</dependencies>
```

RDD接口   
XgboostR.scala
```scala
import org.apache.log4j.{ Level, Logger }
import org.apache.spark.{ SparkConf, SparkContext }
import ml.dmlc.xgboost4j.scala.spark.XGBoost

import org.apache.spark.sql.{ SparkSession, Row }
import org.apache.spark.mllib.util.MLUtils
import org.apache.spark.ml.feature.LabeledPoint
import org.apache.spark.ml.linalg.Vectors

object XgboostR {

  def main(args: Array[String]): Unit = {
    Logger.getLogger("org.apache.spark").setLevel(Level.ERROR)
    Logger.getLogger("org.eclipse.jetty.server").setLevel(Level.OFF)
    val spark = SparkSession.builder.appName("example").
      config("spark.sql.shuffle.partitions", "20").getOrCreate()
    spark.conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")

    val path = "./data/"
    val inputTrainPath = path+ "agaricus.txt.train"
    val inputTestPath = path+ "agaricus.txt.test"
    val train = MLUtils.loadLibSVMFile(spark.sparkContext,inputTrainPath)
    val test = MLUtils.loadLibSVMFile(spark.sparkContext, inputTestPath)
    val traindata = train.map { x =>
      val f = x.features.toArray
      val v = x.label
      LabeledPoint(v, Vectors.dense(f))
    }
    val testdata = test.map { x =>
      val f = x.features.toArray
      val v = x.label
      Vectors.dense(f)
    }

    val numRound = 15
    val paramMap = List(
      "eta" -> 1f,
      "max_depth" ->5, //数的最大深度。缺省值为6 ,取值范围为：[1,∞]
      "silent" -> 1, //取0时表示打印出运行时信息，取1时表示以缄默方式运行，不打印运行时信息。缺省值为0
      "objective" -> "binary:logistic", //定义学习任务及相应的学习目标
      "lambda"->2.5,
      "nthread" -> 1 //XGBoost运行时的线程数。缺省值是当前系统可以获得的最大线程数
    ).toMap
    println(paramMap)

    val model = XGBoost.trainWithRDD(traindata, configMap = paramMap, round = numRound, nWorkers = 55, null, null, useExternalMemory = false, Float.NaN)
    print("************ sucess ********************")

    val result = model.predict(testdata)  // result: org.apache.spark.rdd.RDD[Array[Array[Float]]]
    result.take(10).foreach(println)
    spark.stop()
  }

}
```   

dataFrame接口   
XgboostD.scala       
```scala
import org.apache.log4j.{ Level, Logger }
import org.apache.spark.{ SparkConf, SparkContext }
import ml.dmlc.xgboost4j.scala.spark.XGBoost
import org.apache.spark.mllib.evaluation.BinaryClassificationMetrics
import org.apache.spark.sql.{ SparkSession, Row }

object XgboostD {
  def main(args: Array[String]): Unit = {
    Logger.getLogger("org.apache.spark").setLevel(Level.ERROR)
    Logger.getLogger("org.eclipse.jetty.server").setLevel(Level.OFF)
    val spark = SparkSession.builder.appName("example").
      config("spark.sql.shuffle.partitions", "20").getOrCreate()
    spark.conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
    val path = "./data/"
    val trainString = "agaricus.txt.train"
    val testString = "agaricus.txt.test"

    val train = spark.read.format("libsvm").load(path + trainString).toDF("label", "feature")

    val test = spark.read.format("libsvm").load(path + testString).toDF("label", "feature")

    val numRound = 15

    //"objective" -> "reg:linear", //定义学习任务及相应的学习目标
    //"eval_metric" -> "rmse", //校验数据所需要的评价指标  用于做回归

    val paramMap = List(
      "eta" -> 1f,
      "max_depth" -> 5, //数的最大深度。缺省值为6 ,取值范围为：[1,∞]
      "silent" -> 1, //取0时表示打印出运行时信息，取1时表示以缄默方式运行，不打印运行时信息。缺省值为0
      "objective" -> "binary:logistic", //定义学习任务及相应的学习目标
      "lambda" -> 2.5,
      "nthread" -> 1 //XGBoost运行时的线程数。缺省值是当前系统可以获得的最大线程数
    ).toMap
    val model = XGBoost.trainWithDataFrame(train, paramMap, numRound, 45, obj = null, eval = null, useExternalMemory = false, Float.NaN, "feature", "label")
    val predict = model.transform(test)

    val scoreAndLabels = predict.select(model.getPredictionCol, model.getLabelCol).rdd.map { case Row(score: Double, label: Double) => (score, label) }

    //get the auc
    val metric = new BinaryClassificationMetrics(scoreAndLabels)
    val auc = metric.areaUnderROC()
    println("auc:" + auc)

  }
}
```   

dataFrame接口   
sparkWithDataFrame.scala   
```scala
import ml.dmlc.xgboost4j.scala.Booster
import ml.dmlc.xgboost4j.scala.spark.XGBoost
import org.apache.spark.SparkConf
import org.apache.spark.sql.SparkSession

object sparkWithDataFrame {
  def main(args: Array[String]) {

    val numRound = 100
    val num_workers = 10
    val inputTrainPath = "./data/agaricus.txt.train"
    val inputTestPath = "./data/agaricus.txt.test"

    // 使用kyro序列化，需要对序列化的类进行注册
    val sparkConf = new SparkConf().setAppName("sparkWithDataFrame")
      .set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
    sparkConf.registerKryoClasses(Array(classOf[Booster]))

    val sparkSession = SparkSession.builder().config(sparkConf).getOrCreate()

    val trainDF = sparkSession.sqlContext.read.format("libsvm").load(inputTrainPath)
    val testDF = sparkSession.sqlContext.read.format("libsvm").load(inputTestPath)

    val params = List(
      "eta" -> 0.1f,
      "max_depth" -> 2,
      "objective" -> "binary:logistic"
    ).toMap

    val xgbModel = XGBoost.trainWithDataFrame(trainDF, params, numRound, num_workers, useExternalMemory = true)
    xgbModel.transform(testDF).show()
  }
}
```  

dataFrame接口
CTR.scala
```scala
import collection.mutable.ArrayBuffer
import ml.dmlc.xgboost4j.scala.spark.XGBoost
import org.apache.log4j.{Level, Logger}
import org.apache.spark.ml.feature.LabeledPoint
import org.apache.spark.ml.linalg.{DenseVector, Vector, Vectors}
import org.apache.spark.ml.linalg.SQLDataTypes.VectorType
import org.apache.spark.{SparkConf, SparkContext}
import org.apache.spark.sql.types._
import org.apache.spark.sql.hive.HiveContext
import org.apache.spark.mllib.evaluation.BinaryClassificationMetrics
import org.apache.spark.mllib.evaluation.RegressionMetrics
import org.apache.spark.sql.{Row, SparkSession}
import util.distance

object CTR {
  def main(args: Array[String]): Unit = {
    Logger.getLogger("org.apache.spark").setLevel(Level.ERROR)
    Logger.getLogger("org.eclipse.jetty.server").setLevel(Level.OFF)
    val conf = new SparkConf().setAppName("flowBulkQuery")
    conf.set("spark.sql.codegen", "true")
    conf.set("spark.dynamicallocation.enabled","true")
    conf.set("spark.sql.inMemoryColumnarStorage.compressed", "true")
    conf.set("spark.sql.autoBroadcastJoinThreshold", "2000000")
    conf.set("spark.sql.shuffle.partitions", "3000")
    conf.set("spark.default.parallelism", "3000")
    conf.set("spark.shuffle.consolidateFiles", "true")
    conf.set("mapreduce.job.reduces", "2000")
    conf.set("spark.shuffle.compress", "true")
    conf.set("spark.driver.maxResultSize", "10g")
    conf.set("spark.shuffle.blockTransferService", "nio")
    val sc = new SparkContext(conf)
    sc.setLogLevel("WARN")
    val hiveCtx = new HiveContext(sc)

    val dt ="2018-03-06"
    val dim_type = "app"

    val sql=s"""
	select keyword,shop_id,shop_name,click_pos,kwd_len,editDist,
	ctr,day_pv,day_uv,day_click,day_click_uv,day_orderlines,day_gmv
	from reports.rp_sdl_kwd_shop_feature_CTR_szdata
	where dt ='$dt' and dim_type='$dim_type' and ctr>0 and ctr<1
	limit 10000
	""".stripMargin

    val rowdata = hiveCtx.sql(sql)

    val targetInd = rowdata.columns.indexOf("ctr")
    val ignored = List("ctr","keyword","shop_id","shop_name")
    val featInd = rowdata.columns.diff(ignored).map(rowdata.columns.indexOf(_))

    val schema = new StructType().add("label", DoubleType).add("features",VectorType)

    val data =  rowdata.rdd.map(r =>
      Row(r.getDouble(targetInd), Vectors.dense(featInd.map(r.getDouble(_)).toArray))
    )
    val df = hiveCtx.createDataFrame(data,schema = schema).randomSplit(weights = Array(0.7,0.3),seed=11L)

    val traindata = df(0)
    val testdata = df(1)

    val numRound = 15
    val paramMap = List(
      "eta" -> 1f,
      "max_depth" ->5, //数的最大深度。缺省值为6 ,取值范围为：[1,∞]
      "silent" -> 0, //取0时表示打印出运行时信息，取1时表示以缄默方式运行，不打印运行时信息。缺省值为0
      "objective" -> "reg:linear",  //定义学习任务及相应的学习目标   "objective" -> "binary:logistic",
      "lambda"->2.5,
      "nthread" -> 1 //XGBoost运行时的线程数。缺省值是当前系统可以获得的最大线程数
    ).toMap
    println(paramMap)

    val model = XGBoost.trainWithDataFrame(traindata, params = paramMap, round = numRound, nWorkers = 55,
      obj = null, eval = null, useExternalMemory = false, Float.NaN)

    val predict = model.transform(testdata)
//    val scoreAndLabels = predict.select(model.getPredictionCol, model.getLabelCol).rdd
//      .map{ case Row(score: Array[float], label: Double) => (score(0), label) }

    val scoreAndLabels = predict.select(model.getPredictionCol, model.getLabelCol).rdd
      .map{ row => val r= row.getList(0).toArray;(r(0).toString.toDouble,row.getDouble(1))}


    //get the meanAbsoluteError
    val metric = new RegressionMetrics(scoreAndLabels)
    val meanAbsoluteError = metric.meanAbsoluteError
    println("meanAbsoluteError:" + meanAbsoluteError)
    print("************ sucess ********************")
    predict.show(100)

    //get the auc
//    val metric = new BinaryClassificationMetrics(scoreAndLabels)
//    val auc = metric.areaUnderROC()
//    println("auc:" + auc)
//    val metrics = new RegressionMetrics(scoreAndLabels)
//    println("meanAbsoluteError: " + metrics.meanAbsoluteError)
  }
}
```   


注：从hive表读数据的时候，hive表中的tinyint字段在scala中会被识别为Byte，bigint会被识别为Long   


## Reference  
[1]: [windows下在Java中使用xgboost 详细配置教程](http://blog.csdn.net/eddy_zheng/article/details/51049435)   
[2]: [XGBoost 官网说明](http://xgboost.readthedocs.io/en/latest/jvm/index.html)   
[3]: [旭旭_哥](http://blog.csdn.net/luoyexuge/article/details/71422270)   
[4]: [利用xgboost4j下的xgboost分类模型案例](http://blog.csdn.net/zjwcdd/article/details/78739201)   
[5]: [Spark XGBoost的一些问题](http://blog.csdn.net/jiangda_0_0/article/details/78728551)   
[6]: [XGBoost4J Code Examples](https://github.com/dmlc/xgboost/tree/master/jvm-packages/xgboost4j-example)   
