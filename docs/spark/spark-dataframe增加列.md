<!--
---
title: "spark dataframe 增加列"        
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
# <center>spark dataframe 增加列</center>     

dataframe增加列可以用withColumn函数实现，withColumn函数的第一个参数为增列的列的名字，第二列为dataframe中原来某一列    
`df.withColumn("bb",col("id")*0)`   
"bb"为新增列的名字，"id"为原来的某一列   

如果要新增一列原来的dataframe没有的列，则需要用dataframe的union或者crossJoin来实现     
```scala 
val df = sqlContext.range(0, 10)
val a = sqlContext.createDataFrame(sc.makeRDD(List(Row("abc"))),schema=StructType(List(StructField("ad", StringType, true))))
```
a的列的长度为1，df的列的长度为10，用`crossJoin`拼接 `a.crossJoin(df)`,拼接后第一列的值全部为"abc"    

dataframe的join类似sql的   
```scala
import org.apache.spark.{SparkContext, SparkConf}
import org.apache.spark.sql.SQLContext

case class Persons(id_person: Int, name: String, address: String)
case class Orders(id_order: Int, orderNum: Int, id_person: Int)

object DataFrameTest {
  def main(args: Array[String]) {
    val conf = new SparkConf().setMaster("local[2]").setAppName("DataFrameTest")
    val sc = new SparkContext(conf)

    val sqlContext = new SQLContext(sc)

    val personDataFrame = sqlContext.createDataFrame(List(Persons(1, "张三", "深圳"), Persons(2, "李四", "成都"), Persons(3, "王五", "厦门"), Persons(4, "朱六", "杭州")))
    val orderDataFrame = sqlContext.createDataFrame(List(Orders(1, 325, 2), Orders(2, 34, 3), Orders(3, 533, 1), Orders(4, 444, 1), Orders(5, 777, 11)))

    personDataFrame.join(orderDataFrame, personDataFrame("id_person") === orderDataFrame("id_person")).show()
    personDataFrame.join(orderDataFrame, personDataFrame("id_person") === orderDataFrame("id_person"), "inner").show()
    personDataFrame.join(orderDataFrame, personDataFrame("id_person") === orderDataFrame("id_person"), "left").show()
    personDataFrame.join(orderDataFrame, personDataFrame("id_person") === orderDataFrame("id_person"), "left_outer").show()
    personDataFrame.join(orderDataFrame, personDataFrame("id_person") === orderDataFrame("id_person"), "right").show()
    personDataFrame.join(orderDataFrame, personDataFrame("id_person") === orderDataFrame("id_person"), "right_outer").show()
    personDataFrame.join(orderDataFrame, personDataFrame("id_person") === orderDataFrame("id_person"), "full").show()
    personDataFrame.join(orderDataFrame, personDataFrame("id_person") === orderDataFrame("id_person"), "full_outer").show()
    personDataFrame.join(orderDataFrame, personDataFrame("id_person") === orderDataFrame("id_person"), "outer").show()
    personDataFrame.join(orderDataFrame, Seq("id_person"), "left_outer").show()  //避免重复列
  }
}
```



```scala
object JoinType {
  def apply(typ: String): JoinType = typ.toLowerCase.replace("_", "") match {
    case "inner" => Inner
    case "outer" | "full" | "fullouter" => FullOuter
    case "leftouter" | "left" => LeftOuter
    case "rightouter" | "right" => RightOuter
    case "leftsemi" => LeftSemi
    case _ =>
      val supported = Seq(
        "inner",
        "outer", "full", "fullouter",
        "leftouter", "left",
        "rightouter", "right",
        "leftsemi")

      throw new IllegalArgumentException(s"Unsupported join type '$typ'. " +
        "Supported join types include: " + supported.mkString("'", "', '", "'") + ".")
  }
}
```

sealed abstract class JoinType

case object Inner extends JoinType

case object LeftOuter extends JoinType

case object RightOuter extends JoinType

case object FullOuter extends JoinType

case object LeftSemi extends JoinType


refer:    
1. http://blog.csdn.net/sparkexpert/article/details/51023375    
2. http://blog.csdn.net/anjingwunai/article/details/51934921    
3. http://blog.csdn.net/sparkexpert/article/details/52837269    