<!--
---
title: idea初始化配置 
author: styleNie
date: 2018-08-29
tags: IDEA

toc:
    depth_from: 1
    depth_to: 6
    ordered: false
    ignoreLink: false

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
## 修改IntelliJ IDEA系统缓存目录      
在IDEA安装目录的bin文件夹中找到idea.properties文件，将idea.config.path和idea.system.path改成将要存放目录的位置    
```html   
idea.config.path=D:/software/IDEA/IDEAConfig/config
idea.system.path=D:/software/IDEA/IDEAConfig/system
idea.plugins.path=D:/software/IDEA/IDEAConfig/plugins
idea.log.path=D:/software/IDEA/IDEAConfig/log
```    

## 修改SBT的目录路径和ivy的目录仓库地址    
进入安装sbt\conf目录，打开sbtconfig.txt文件，如下设置    
```html
-Dsbt.global.base=D:/TOOL_TEM/sbt/.sbt
-Dsbt.ivy.home=D:/TOOL_TEM/sbt/.ivy2
```     

## 修改maven仓库目录    
在IDEA自带的maven插件目录下找到maven的settings.xml,一般在  IntelliJ IDEA\plugins\maven\lib  目录下，会有maven2和maven3两个目录，在conf目录下都有settings，修改localRepository属性         
```html    
<localRepository>D:/mavenrepository</localRepository>
```      


Reference:    
[[至逝去的一年]修改IntelliJ IDEA修改系统缓存目录,修改sbt的.sbt和.ivy2](https://www.cnblogs.com/artisanWay/p/5602266.html)     
[IntelliJ IDEA 详细图解最常用的配置 ，适合刚刚用的新人](https://blog.csdn.net/qq_27093465/article/details/52918873)    
[在idea中maven项目 jar包下载不完整解决办法](https://blog.csdn.net/qq_34579313/article/details/80957342)


