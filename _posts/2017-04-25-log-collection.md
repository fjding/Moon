---
layout: post
title: "Docker容器日志收集"
date: 2017-04-25
excerpt: "针对Docker容器日志收集的方案的分析，以及各种收集方案的设计实现方案."
tags: [Docker, 日志收集]
comments: true
---



### 技术难点 ###
&ensp;&ensp;&ensp;&ensp;对于Docker来说，各个docker容器是相互隔离的，因此在容器中引用程序产生的日志系统也是分布在各个容器的文件存储系统中，在宿主主机中无法看到容器中的各个文件，当有多个容器的时候，一个日志收集器无法对各个容器中日志文件进行收集。


### Docker相关技术 ###
1. Docker Data Volume（数据卷）
2. Docker Data Volume Container（数据卷容器）


### 解决方案 ###
方案1：在每一个容器中运行一个日志收集器；【缺点：容器较多时收集器增多，消耗系统资源】<br>
方案2：利用Data Volume的-v技术，将各个容器中日志文件挂载到宿主主机目录中，启动一个日志收集器进程（或者收集器容器）对日志进行收集。<br>
方案3：利用Data Volume的volumes-from技术，运行一个日志收集器容器，利用容器间文件共享实现对各个容器中日志文件收集。

### 详细方案 ###
方案1：结构图如图1所示<br>

<figure>
	<img src="{{ site.url }}/assets/myimg/log-collection/20170425161901.png">
</figure>

<center> 图1 方案1结构图 </center>
&ensp;&ensp;&ensp;&ensp;每个主机中一个运行多个docker容器，每个容器运行一个或多个服务，产生的日志可以通过日志收集器的agent进行收集，将收集日志发送到中间件或者存储系统。<br>
&ensp;&ensp;&ensp;&ensp;该实现比较简单，对于某些场景来说经常采用，比如采用flume收集exec_source进行收集的时候，当日志文件中有更新的时候，exec_source就会读取新增加的日志，该方案是比较合适。


方案2：结构如图2所示<br>
![]({{ site.url }}/assets/myimg/log-collection/20170425161924.png)
<center> 图2 方案2结构图 </center>
&ensp;&ensp;&ensp;&ensp;该种方案主要是将日志收集器部署在docker容器的宿主主机上，通过volume -v技术将根据一定的存储策略将每个容器中共享到宿主主机的log主目录下，然后日志收集器定期收集该目录下的文件。<br>
&ensp;&ensp;&ensp;&ensp;假设要在一台主机上运行两个容器，容器中运行的应用都是Tomcat,（建议一个容器中指部署一个同类应用服务），则最佳的方案是运行一个shell脚本或者一个Dockerfile就能创建两个容器，每个容器中运行一个Tomcat服务，而且保证在宿主主机的日志收集目录/log/的子目录下创建不同的子目录名称，以区分不同容器的Tomcat产生的日志。

方案3：结构如图3如所示 <br>
<figure>
	<img src="{{ site.url }}/assets/myimg/log-collection/20170425162126.png">
</figure>
<center> 图3 方案3结构图 </center>
&ensp;&ensp;&ensp;&ensp;该方案与方案2的区别是直接在日志收集容器中创建一个共享文件目录，然后将共享目录与其他各个容器进行共享，而不是将日志收集器直接安装在宿主主机进行监控。一般采用该方式收集比第二种收集更加优雅。

### 总结 ###
&ensp;&ensp;&ensp;&ensp;以上三种方案是根据项目中的一些需求提出的，在构建一个大型的日志收集平台的时候，需要根据应用场景组合多种方式进行收集，总体来说第一种方案用得比较多，但是当遇到容器中有多个日志文件需要实时监控时，为了更少的依赖其他的容器，方便自动化部署，直接将收集器部署在运行应用服务的容器比较合适的。
