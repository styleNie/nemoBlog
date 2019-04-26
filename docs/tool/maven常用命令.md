<!--
---
title: maven常用命令 
author: styleNie
date: 2018-08-29
tags: maven

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
Maven库：

```http://repo2.maven.org/maven2/```

Maven依赖查询：

```http://mvnrepository.com/```

Maven常用命令： 
```
1. 创建Maven的普通java项目： 
   mvn archetype:create 
   -DgroupId=packageName 
   -DartifactId=projectName  
2. 创建Maven的Web项目：   
    mvn archetype:create 
    -DgroupId=packageName    
    -DartifactId=webappName 
    -DarchetypeArtifactId=maven-archetype-webapp    
3. 编译源代码： mvn compile 
4. 编译测试代码：mvn test-compile    
5. 运行测试：mvn test   
6. 产生site：mvn site   
7. 打包：mvn package   
8. 在本地Repository中安装jar：mvn install 
9. 清除产生的项目：mvn clean   
10. 生成eclipse项目：mvn eclipse:eclipse  
11. 生成idea项目：mvn idea:idea  
12. 组合使用goal命令，如只打包不测试：mvn -Dtest package   
13. 编译测试的内容：mvn test-compile  
14. 只打jar包: mvn jar:jar  
15. 只测试而不编译，也不测试编译：mvn test -skipping compile -skipping test-compile 
      ( -skipping 的灵活运用，当然也可以用于其他组合命令)  
16. 清除eclipse的一些系统设置:mvn eclipse:clean     
17. 加载本地jar到本地库中 mvn install:install-file -Dfile=C:\Users\admin\Downloads\kaptcha-2.3.jar -DgroupId=com.google.code  -DartifactId=kaptcha -Dversion=2.3 -Dpackaging=jar
```


Reference:   
[IDEA中常用的maven指令](https://blog.csdn.net/u012031380/article/details/53584858)
