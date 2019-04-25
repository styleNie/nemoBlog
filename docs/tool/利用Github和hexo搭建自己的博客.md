---
title: 利用 Github 和 hexo 搭建自己的博客
author: styleNie
date: 2018-07-19 16:33:02
tags: hexo

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

## 环境准备   
- [Github](https://github.com/)账号   
- [Git](https://github.com/waylau/git-for-win)客户端，[淘宝镜像](https://npm.taobao.org/mirrors/git-for-windows/2.12.0.windows.1/Git-2.12.0-64-bit.exe),版本2.12      
- [node.js](https://nodejs.org/en/),版本10.7   
- windows10   

## 搭建博客   
### Github配置   
进入Github点击新建库，在创建页面填入域名，域名格式： username.github.io   

### Git配置    
打开安装好的git, 使用git bash   
执行命令1：```git config --global user.name [username] ```  
比如我的就是：```git config --global user.name styleNie ``` 

执行命令2：```git config --global user.email [你的github邮箱名]```   

执行命令3：```ssh-keygen -t rsa -C user.email [你的github邮箱名]```   
然后一直enter 3下（这里会问你是否需要密码，是否确定等，略过就好，不用太操心）
然后在 ```C:\Users\administractor.ssh``` 下找到文件：```id_rsa.pub``` 用记事本打开，复制里面的内容。
回到 GitHub设置，```Add SSH key```，粘贴进去。
到此github部分结束.    

## node.js & hexo    
### 配置 hexo     
node.js安装完成后，   
输入指令1：```npm install -g hexo-cli```   
这里会比比较慢，等等就好。完成后继续输入   
输入指令2：```hexo init```   
输入指令3：```hexo install```   
完成后会有：```_config.yml```、```package.json```等等的文件和文件夹    

测试下是否可用      
输入指令1：```hexo generate```   
输入指令2：```hexo server```   
这样本地已经可以浏览了，可以去```localhost:4000``` 查看。你会看到一份```hello world``` 的文章，这个文章是放在你刚刚创建的那个文件夹的 ```source/_posts``` 这个文件夹下    

### 部署到网络上   
为了能发布到Github上，需要先安装一个插件      
```html
 npm install hexo-deployer-git --save
```   

在你创建的文件夹下找到 ```_config.yml``` 并打开，修改最后一段。注意输入这些的时候一定要是英文，且冒号之后必须有空格。不然你输入命令的时候会没有反应   

```html
deploy: 
  type: git
  repository: ssh://git@github.com/styleNie/styleNie.github.io
  branch: masters
```    
在新建的文件夹下打开git bash    
输入命令1：```hexo generate```  
输入命令2：```hexo deploy```      
最后输出 ```[info] Deploy done: git``` 表示成功   
上网站： ```username.github.io``` 查看新发布的文章。    

### 发布新文章   
cmd 输入命令 ```hexo new "新文章"``` 然后去```source/_post``` 文件夹下就可以看到一个 ```新文章.md``` 的文件，用markdown格式编写完成后，git bash下输入以下命令   

```html
hexo clean   
hexo generate   
hexo deploy
```      

## 主题设置   
参考[NexT](http://theme-next.iissnan.com/getting-started.html#menu-settings) 的使用。   
### 安装 NexT   
在[部署到网络上](部署到网络上) 中新建的文件夹下， git clone https://github.com/iissnan/hexo-theme-next themes/next   

### 启用主题   
打开站点配置文件 ```_config.yml```，找到```theme```字段，并将其改为```next```。   

更多主题设置尽在 NexT。

Reference:   
1. [zealotCE](https://zealotce.github.io/2017/03/03/build_your_own_blog%20with%20git+hexo/)   
2. [hexo无法上传到github](https://segmentfault.com/q/1010000003734223)    
3. [NexT使用文档](http://theme-next.iissnan.com/getting-started.html#menu-settings)      
4. [GitHub Pages + Hexo搭建博客](http://crazymilk.github.io/2015/12/28/GitHub-Pages-Hexo%E6%90%AD%E5%BB%BA%E5%8D%9A%E5%AE%A2/#more)      
5. [](http://www.jeyzhang.com/hexo-next-add-post-views.html)


https://github.com/theme-next/hexo-theme-next/issues/461

https://www.jianshu.com/p/fea9fc962005

