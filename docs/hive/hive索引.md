<!--
---
title: "hive索引"      
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
# <center>hive索引</center>
<center> <font face="黑体">目录</font></center>     

[TOC]    

## 建索引操作步骤    
- 索引配置信息    
```html
set hive.exec.dynamic.partition.mode=nonstrict;  ----设置所有列为 dynamic partition
set hive.exec.dynamic.partition=true;                   ----使用动态分区
set hive.optimize.index.filter=false;
set hive.optimize.index.groupby=false;
```
hive.optimize.index.filter 和 hive.optimize.index.groupby 参数默认是 false。使用索引的时候必须把这两个参数开启，才能起到作用。      
hive.optimize.index.filter.compact.minsize   参数为输入一个紧凑的索引将被自动采用最小尺寸、默认5368709120(以字节为单位)      

- 建索引
```html
create index brandname_index on table mytable(brandname_cn)
as 'org.apache.hadoop.hive.ql.index.compact.CompactIndexHandler' with deferred rebuild;
```
brandname_index 为索引名，在 brandname_cn 这个字段建的索引    

- 更新索引
```html
alter index brandname_index on mytable rebuild;
```
这个步骤表中的数据越多，花的时间越长    

- 删除索引
```html
DROP INDEX brandname_index on mytable;
```

- 查看索引
```html
show index on  mytable
```
      
## presto与hive索引    
```html
Presto本身不支持建索引，但是建了索引的hive表用presto也能查。
```    

## hive分桶       
- 桶的概念    
对于每一个表(table)或者分区， Hive可以进一步组织成桶，也就是说桶是更为细粒度的数据范围划分。Hive也是针对某一列进行桶的组织。Hive采用对列值哈希，然后除以桶的个数求余的方式决定该条记录存放在哪个桶当中。把表(或者分区)组织成桶(Bucket)有两个理由：   
(1) 获得更高的查询处理效率。桶为表加上了额外的结构，Hive 在处理有些查询时能利用这个结构。具体而言，连接两个在(包含连接列的)相同列上划分了桶的表，可以使用 Map 端连接(Map-side join)高效的实现。比如JOIN操作。对于JOIN操作两个表有一个相同的列，如果对这两个表都进行了桶操作。那么将保存相同列值的桶进行JOIN操作就可以，可以大大较少JOIN的数据量。   
(2) 使取样(sampling)更高效。在处理大规模数据集时，在开发和修改查询的阶段，如果能在数据集的一小部分数据上试运行查询，会带来很多方便。

- hive桶操作步骤   
设置环境变量
```html
set hive.enforce.bucketing=true;
set hive.enforce.sorting=true;
```    

```html
create table if not exists mytable(
brandname_cn string comment '品牌名',
key_word string comment '关键词'
)partitioned by (dt string)
clustered by(brandname_cn) into 5 buckets
stored as orc tblproperties('orc.compress'='SNAPPY');
```    

hive支持先分区再分桶，用brandname_cn做分桶字段
插入数据时不需要其它特别操作。

Hive的分区表存储时会按分区字段组织，比如上面的表每个日期一个文件，但是分桶不会。   



## presto与hive分桶   
presto不支持分桶，hive的分桶表用presto查询会报错   
```html
Hive table is corrupt. It is declared as being bucketed, but the files do not match the bucketing declaration. The number of files in the directory (4) does not match the declared bucket count (5) for partition: dim_type=qq/dt=2017-09-03
```   

## Reference   
1  [hive索引](http://www.cnblogs.com/zlslch/p/6105294.html)     
2  [presto社区关于索引的讨论](https://groups.google.com/forum/#!msg/presto-users/ab3BKCkYf8k/kfdJnZXTFgAJ;context-place=forum/presto-users)


