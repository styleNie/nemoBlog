<!--
---
title: "Spark读写hbase"    
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
# <center>Spark读写hbase</center>
  

## 	数据准备   
- Hbase 版本1.1.6
- Spark版本 2.1.0 

- Hive导数据到hbase格式：   
```sql
select concat(md5(key_word), regexp_replace(dt,'-','')) as rowkey, 
key_word, dt
from mytable
where dt >= sysdate(-1)
```    
- 以key_word的md5和日期拼接的字符串作为rowkey   

## 	按rowkey过滤条件读取Hbase     
```scala
import org.apache.hadoop.hbase.HBaseConfiguration
import org.apache.hadoop.hbase.mapreduce.TableInputFormat
import org.apache.spark._
import org.apache.hadoop.hbase.util.Bytes

import org.apache.hadoop.hbase.client.Scan
import org.apache.hadoop.hbase.filter.{CompareFilter, Filter, RegexStringComparator, RowFilter}
import org.apache.hadoop.hbase.util.Base64
import org.apache.hadoop.hbase.mapreduce.TableMapReduceUtil
import org.apache.hadoop.hbase.protobuf.ProtobufUtil

object readHbase2 {

  def main(args: Array[String]): Unit = {
    val sparkConf = new SparkConf().setAppName("HBaseTest")
    sparkConf.set("spark.default.parallelism", "1000")
    sparkConf.set("spark.shuffle.consolidateFiles", "true")
    sparkConf.set("mapreduce.job.reduces", "1000")
    sparkConf.set("spark.shuffle.compress", "true")
    val sc = new SparkContext(sparkConf)
    //    val conf = new HBaseConfiguration()
    //    val hbaseContext = new HBaseContext(sc, config)
    val tablename = ""
    val conf = HBaseConfiguration.create()
    //设置zooKeeper集群地址，也可以通过将hbase-site.xml导入classpath，但是建议在程序里这样设置
    conf.set("hbase.zookeeper.quorum","192.168.1.100")
    //baseZNode=/hbase_hades
    //设置zookeeper连接端口，默认2181
    conf.set("hbase.zookeeper.property.clientPort", "2181")
    conf.set("zookeeper.znode.parent","")
    conf.set("bdp.hbase.erp", "") //你的erp
    conf.set("bdp.hbase.instance.name", "") //申请的实例名称
    conf.set("bdp.hbase.accesskey", "") //实例对应的accesskey，请妥善保管你的AccessKey

//    val startRowkey= new String(MD5Util.getMD5("动漫挂件").concat("20171001").toLowerCase)
//    val endRowkey=new String(MD5Util.getMD5("动漫挂件").concat("20171017").toLowerCase)
    // co RowFilterExample-2-Filter2 Another filter, this time using a regular expression to match the row keys.  new RegexStringComparator("\".*20171015\"")
    val filter2 = new RowFilter(CompareFilter.CompareOp.EQUAL, new RegexStringComparator(".*20171015$"))   //正则匹配后缀过滤，取2017-10-15日的数据，hbase本身提供了前缀过滤器PrefixFilter

    val scan=new Scan()
    scan.setCacheBlocks(false)
    scan.setCaching(10000)           // 设置缓存数据条数
    scan.setFilter(filter2)
    scan.addFamily(Bytes.toBytes("d"))
    scan.addColumn(Bytes.toBytes("d"), Bytes.toBytes("key_word"))
    scan.addColumn(Bytes.toBytes("d"), Bytes.toBytes("dt"))

    //将scan类转化成string类型
    val scan_str= convertScanToString(scan)    //这个转换函数在TableMapReduceUtil类中，但是没法直接调用，这里把这个函数直接复制过来
    conf.set(TableInputFormat.SCAN,scan_str)
    conf.set(TableInputFormat.SCAN_CACHEDROWS,"1000")
    conf.set(TableInputFormat.INPUT_TABLE, tablename)

    //读取数据并转化成rdd
    val hBaseRDD = sc.newAPIHadoopRDD(conf, classOf[TableInputFormat],
      classOf[org.apache.hadoop.hbase.io.ImmutableBytesWritable],
      classOf[org.apache.hadoop.hbase.client.Result])
//  读取出来的RDD的分区数目由hbase表的分区数决定
    val a=hBaseRDD.take(10)
  println(a.foreach(x=>println(x)))
//  读取出来的数据需要转换一下格式，全部用的toString函数转换，避免hbase中数据类型和spark的数据类型不兼容而报错  java.io.IOException: java.io.IOException: java.lang.IllegalArgumentException: offset (0) + length (8) exceed the capacity of the array: 4
    val rd=hBaseRDD.map(x =>{
      val result=x._2
      //获取行键
      val key = Bytes.toString(result.getRow)
      //通过列族和列名获取列
      val key_word = Bytes.toString(result.getValue("d".getBytes,"key_word".getBytes))
      val dt=Bytes.toString(result.getValue("d".getBytes,"dt".getBytes))
      (key_word,dt)
    })
    println("***************  ")
    rd.take(10).foreach(x=>println(x))
    println("------------------------"+hBaseRDD.getNumPartitions)
    //println("***************  "+rd.count())

    sc.stop()
  }

  def convertScanToString(scan: Scan) = {
    val proto = ProtobufUtil.toScan(scan)
    Base64.encodeBytes(proto.toByteArray)
  }
}
```   
   
##  spark写数据到Hbase 
```scala
import org.apache.hadoop.hbase.{HBaseConfiguration, TableName}
import org.apache.hadoop.hbase.client.{ConnectionFactory, Put}
import org.apache.hadoop.hbase.io.ImmutableBytesWritable
import org.apache.hadoop.hbase.mapred.TableOutputFormat
import org.apache.hadoop.hbase.util.Bytes
import org.apache.hadoop.mapred.JobConf
import org.apache.spark.sql.hive
import org.apache.spark.{SparkConf, SparkContext}


object writeHbase2 {
  def main(args: Array[String]): Unit = {
    val sc = new SparkContext(new SparkConf().setAppName("writeHbase2"))
    val conf = HBaseConfiguration.create()
    val tablename = ""
    val HB_FAMILY:String = "d"
    var jobConf = new JobConf(conf)
    jobConf.set("hbase.zookeeper.quorum","192.168.1.100")
    //设置zookeeper连接端口，默认2181
    jobConf.set("hbase.zookeeper.property.clientPort", "2181")
    jobConf.set("zookeeper.znode.parent","")
    jobConf.set("bdp.hbase.erp", "") //你的
    jobConf.set("bdp.hbase.instance.name", "") //申请的实例名称
    jobConf.set("bdp.hbase.accesskey", "") //实例对应的accesskey，请妥善保管你的AccessKey
    jobConf.set(TableOutputFormat.OUTPUT_TABLE, tablename)
    jobConf.setOutputFormat(classOf[TableOutputFormat])

    val sqlStr="""
              select concat(md5(key_word), regexp_replace(dt,'-','')) as rowkey,
              key_word, dt
              from mytable
              where dt ='2017-10-17'
    		"""

    val hiveCtx = new hive.HiveContext(sc)
    val keyRdd = hiveCtx.sql(sqlStr).rdd
    val start=System.currentTimeMillis()

    keyRdd.map(row=>{
      var field=row.mkString(",").split(",")
      var key= field(0)
      var put = new Put(Bytes.toBytes(key))
      put.addColumn(Bytes.toBytes("d"), Bytes.toBytes("key_word"), Bytes.toBytes(field(1)))
      put.addColumn(Bytes.toBytes("d"), Bytes.toBytes("dt"), Bytes.toBytes(field(12)))
      println("handle data end!!!")
      (new ImmutableBytesWritable, put)
    }).saveAsHadoopDataset(jobConf)

    val end=System.currentTimeMillis()
    System.out.println("******************* cost time: "+(end-start)/1000+" s")
  }
}
```   


- Pom文件配置   
```html
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com</groupId>
    <artifactId>data</artifactId>
    <version>1.0-SNAPSHOT</version>
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <scala.version>2.11.8</scala.version>

        <!--add  maven release-->
        <maven.compiler.source>1.7</maven.compiler.source>
        <maven.compiler.target>1.7</maven.compiler.target>
        <encoding>UTF-8</encoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.apache.hbase</groupId>
            <artifactId>sdk</artifactId>
            <version>2.1-SNAPSHOT</version>
            <!--<scope>provided</scope>-->
        </dependency>
        <dependency>
            <groupId>org.apache.hbase</groupId>
            <artifactId>hbase-common</artifactId>
            <version>1.1.6</version>
            <!--<scope>provided</scope>-->
        </dependency>
        <dependency>
            <groupId>org.apache.hbase</groupId>
            <artifactId>hbase-server</artifactId>
            <version>1.1.6</version>
            <!--<scope>provided</scope>-->
        </dependency>
        <dependency>
            <groupId>org.apache.hbase</groupId>
            <artifactId>hbase-client</artifactId>
            <version>1.1.6</version>
            <!--<scope>provided</scope>-->
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_2.11</artifactId>
            <version>2.1.0</version>
            <scope>provided</scope>
        </dependency>

        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-sql_2.11</artifactId>
            <version>2.1.0</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-hive_2.11</artifactId>
            <version>2.1.0</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>

    <build>
        <finalName>athenaUmpHbase</finalName>
        <sourceDirectory>src/main/scala</sourceDirectory>
        <!--<testSourceDirectory>src/test/scala</testSourceDirectory>-->
        <plugins>
            <plugin>
                <groupId>org.scala-tools</groupId>
                <artifactId>maven-scala-plugin</artifactId>
                <executions>
                    <execution>
                        <goals>
                            <goal>compile</goal>
                            <goal>testCompile</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <scalaVersion>${scala.version}</scalaVersion>
                    <args>
                        <arg>-target:jvm-1.7</arg>
                    </args>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-eclipse-plugin</artifactId>
                <configuration>
                    <downloadSources>true</downloadSources>
                    <buildcommands>
                        <buildcommand>ch.epfl.lamp.sdt.core.scalabuilder</buildcommand>
                    </buildcommands>
                    <additionalProjectnatures>
                        <projectnature>ch.epfl.lamp.sdt.core.scalanature</projectnature>
                    </additionalProjectnatures>
                    <classpathContainers>
                        <classpathContainer>org.eclipse.jdt.launching.JRE_CONTAINER</classpathContainer>
                        <classpathContainer>ch.epfl.lamp.sdt.launching.SCALA_CONTAINER</classpathContainer>
                    </classpathContainers>
                </configuration>
            </plugin>
            <plugin>
                <artifactId>maven-assembly-plugin</artifactId>
                <configuration>
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>
                    <archive>
                        <manifest>
                            <mainClass>readHbase2</mainClass>
                        </manifest>
                    </archive>
                </configuration>
                <executions>
                    <execution>
                        <id>make-assembly</id>
                        <phase>package</phase>
                        <goals>
                            <goal>single</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
    <reporting>
        <plugins>
            <plugin>
                <groupId>org.scala-tools</groupId>
                <artifactId>maven-scala-plugin</artifactId>
                <configuration>
                    <scalaVersion>${scala.version}</scalaVersion>
                </configuration>
            </plugin>
        </plugins>
    </reporting>
</project>
```   

Reference:
1.	http://www.cnblogs.com/shenguanpu/archive/2012/06/12/2546309.html   
2.	http://blog.csdn.net/Gpwner/article/details/73530134   
3.	https://www.iteblog.com/archives/1892.html   



