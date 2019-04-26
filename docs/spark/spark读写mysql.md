<!--
---
title: "spark读写mysql"      
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
# <center>spark读写mysql</center>    
spark版本2.1    
```html
<dependency>
<groupId>org.apache.spark</groupId>
<artifactId>spark-sql_2.11</artifactId>
<version>${spark.version}</version>
<!--<scope>provided</scope>-->
</dependency>
<dependency>
<groupId>org.apache.spark</groupId>
<artifactId>spark-hive_2.11</artifactId>
<version>${spark.version}</version>
<!--<scope>provided</scope>-->
</dependency>
<!-- https://mvnrepository.com/artifact/mysql/mysql-connector-java -->
<dependency>
<groupId>mysql</groupId>
<artifactId>mysql-connector-java</artifactId>
<version>5.1.31</version>
</dependency>
```    

读mysql   
```scala
import org.apache.spark.{SparkConf, SparkContext}
import org.apache.spark.sql.{SQLContext, SaveMode}
import java.util.Properties

/**
  * Created by mi on 17-4-11.
  */

case class resultset(name: String,info: String,summary: String)

object MysqlOpt {
  def main(args: Array[String]): Unit = {
    val conf = new SparkConf().setAppName("WordCount").setMaster("local")
    val sc = new SparkContext(conf)
    val sqlContext = new SQLContext(sc)
    import sqlContext.implicits._

    //定义数据库和表信息
    val url = "jdbc:mysql://localhost:3306/baidubaike?useUnicode=true&characterEncoding=UTF-8"
    val table = "baike_pages"

    //读MySQL的方法1
    val reader = sqlContext.read.format("jdbc")
    reader.option("url", url)
    reader.option("dbtable", table)
    reader.option("driver", "com.mysql.jdbc.Driver")
    reader.option("user", "root")
    reader.option("password", "XXX")
    val df = reader.load()
    df.show()

    //读MySQL的方法2
    //    val jdbcDF = sqlContext.read.format("jdbc").options(
    //      Map("url"->"jdbc:mysql://localhost:3306/baidubaike?useUnicode=true&characterEncoding=UTF-8",
    //        "dbtable"->"(select name,info,summary from baike_pages) as some_alias",
    //        "driver"->"com.mysql.jdbc.Driver",
    //        "user"-> "root",
    //        //"partitionColumn"->"day_id",
    //        "lowerBound"->"0",
    //        "upperBound"-> "1000",
    //        //"numPartitions"->"2",
    //        "fetchSize"->"100",
    //        "password"->"XXX")).load()
    //    jdbcDF.show()

  }
}
```    

写 mysql   
```scala
import org.apache.spark.{SparkConf, SparkContext}
import org.apache.spark.sql.{SQLContext, SaveMode}
import java.util.Properties

/**
  * Created by mi on 17-4-11.
  */
case class resultset(name: String,info: String,summary: String)
object MysqlOpt {
  def main(args: Array[String]): Unit = {
    val conf = new SparkConf().setAppName("WordCount").setMaster("local")
    val sc = new SparkContext(conf)
    val sqlContext = new SQLContext(sc)
    import sqlContext.implicits._

    //定义数据库和表信息
    val url = "jdbc:mysql://localhost:3306/baidubaike?useUnicode=true&characterEncoding=UTF-8"
    val table = "baike_pages"

    //写MySQL的方法1
    val list = List(
      resultset("名字1", "标题1", "简介1"),
      resultset("名字2", "标题2", "简介2"),
      resultset("名字3", "标题3", "简介3"),
      resultset("名字4", "标题4", "简介4")
    )
    val jdbcDF = sqlContext.createDataFrame(list)
    jdbcDF.collect().take(20).foreach(println)
    //    jdbcDF.rdd.saveAsTextFile("/home/mi/coding/coding/Scala/spark-hbase/output")
    val prop = new Properties()
    prop.setProperty("user", "root")
    prop.setProperty("password", "123456")
    //jdbcDF.write.mode(SaveMode.Overwrite).jdbc(url,"baike_pages",prop)
    jdbcDF.write.mode(SaveMode.Append).jdbc(url, "baike_pages", prop)
  }
}
```


refer:    
1. http://www.cnblogs.com/tonglin0325/p/6702518.html