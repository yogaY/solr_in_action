# Solr_in_Action 2.1

标签： 翻译 solr_in_action

作业部落编写，[zybuluo](https://www.zybuluo.com/tsihedeyo/note/25071)，可见图片

[TOC]

---

# Solr_in_Actin 第2章

此章涉及

> * 下载及安装Apache Solr 4.7
> * 启动Solr服务器
> * 排序、分页、结果格式化
> * 探索Solritas示例搜索UI

刚上手一个新的技术肯定会不适应，但对Solr你可以放心，因为Solr就是很容易安装部署。为了保持灵活性，你可以从基本的Solr配置开始，逐渐增加再复杂化你的配置。例如，Solr支持将一个大的索引分割成很多小的子集(称为shards)，也支持索引重复来增强查询的并发处理，但你不需要急着配置这些，用到再说。

读完这章，你应该已经在服务器上运行Solr，知道如何启动停止、使用页面管理控制台并且对Solr的关键术语都有了基本的了解（Solr home、core、collection）。

> *扩展*
> #### 名字意义？ Solr 4 vs. SolrCloud
> 略

---
## 2.1. 开始

在了解Solr之前，你必须跑通Solr。第一步，从Apache下载Solr4.7源码并解压。安装完毕，我们将演示如何启动Solr服务并访问web管理页面管理。整个过程中希望你能接受在你的OS里使用命令行工作，因为Sorl没有可视化的安装，即便如此，安装还是十分简单以至于根本就不需要可视化安装。

---

### 2.1.1. 安装Solr

由于你需要做的就是下载源码，解压，所以都不能把这称之为安装。在这之前，确保你的Java版本大于1.6，命令行确认Java 版本

    java -version
    
你会看到如下类似如下输出

    java version "1.6.0_24"
    Java™ SE Runtime Environment (build 1.6.0_24-b07)
    Java HotSpot™ 64-Bit Server VM (build 19.1-b02, mixed mode)
    
若你没有安装Java，推荐安装Oracel的JVM（www.oracle.com/technetwork/java/javase/downloads/index.html）。
安装完Java之后，安装Solr。

去Solr主页下载最新的Solr（http://lucene.apache.org/solr）。若在windows下，下载sorl-4.7.0.zip。若Unix、Linux、Mac OS，下载solr-4.7.0.tgz。本书中所有的例子都是基于4.7.0，若4.7.0已经不在下载页，你可以去这里http://archive.apache.org/dist/lucene/solr/4.7.0/查找solr4.7.0下载。

省略。

加压后源码包目录结构 如图2.1

**图2.1 solr-4.7.0源码结构，本书中所有的顶层目录为$SOLR_INSTALL/**
![图2.1] (https://raw.githubusercontent.com/yyjtry/solr_in_action/master/solr_chapter_1/1.4.png)

你解压源码的目录为`$SOLR_INSTALL`，`$SOLOR_HOME`另有所指，所以不用这个名字。

---

### 2.1.2. 开启Solr服务器

打开命令行，输入

    cd $SOLR_INSTALL/example
    java -jar start.jar
    
运行过程中，控制台会有输出信息，若顺利，你会看到如下信息
    
    3504 [main] INFO org.eclipse.jetty.server.AbstractConnector – Started SocketConnector@0.0.0.0:8983

####**发生了什么**
 
 安装太简单了你可能都要怀疑是不是安装完了，相信我现在在你的电脑上已经跑着个Solr4.7了。在浏览器输入
 http://localhost:8983/solr进行验证。图2.2是Solr 管理员控制台的一个快照，瞄一眼习惯下布局以及控制台的一些工具。
 
 **图2.2 Solr 4.7 管理员控制界面，提供了丰富的工具。点击Collection1链接获取更多工具，这其中就包括了搜索表单。**
![图2.2](https://raw.githubusercontent.com/yyjtry/solr_in_action/master/solr_chapter_1/1.5.png)
 
在后台，start.jar启动了Jetty服务器，监听了8983端口。Solr就是个运行在Jetty内的web应用。图2.3显示了你电脑上运行的程序。

**图2.3 在系统看来，Solr是个基于Java的Jetty服务器上的一个Solr web应用(solr.war)。每一个Jetty服务器上有个Solr home 目录集，按照Java的特征为solr.solr.home。Solr的一台服务器可以支持多个cores，而每个core在home目录下有单独的目录（如collection1）,其中包括了core-specfic配置以及索引（data）。**
![图2.3](https://raw.githubusercontent.com/yyjtry/solr_in_action/master/solr_chapter_2/2.3.png)


####**排错**

开启服务器基本有什么错，最普遍的错误就是8983端口被别的程序占用而使得无法启动。这种情况下，你会看到反馈报错`java.net.BindException:Address already in use`。解决这个问题就是在启动命令行时指定端口号，使用命令,如下指定端口8080

    java -Djetty.port=8080 -jar start.jar
    
> *扩展*
> **Jetty vs. Tomcat**
> 略

####**关闭Solr**

本地操作中，控制台输入Ctrl-c关闭Solr。开发和测试中可以这样，Jetty提供了更安全的机制去停止服务器，在章节12中会讨论。

既然Solr已经跑起来了，下面去理解Solr从哪里获取的配置信息，它又是如何管理Lucene索引的。

---

###2.1.3 理解Solr home

在Solr中，一个core有一堆配置文件、Lucenen 索引文件、Solr的事务日志组成。Jetty中跑的一个Solr服务器可以支持多个cores。回想在第一章我们设计的房产搜索应用，它一个core处理房子，另一个core处理土地信息。由于这两种索引信息差别大到我们需要构建两种不同的索引，所以我们开了两个core。我们在2.1.2中启动的示例服务器只有一个叫collection1的core。

声明：Solrs使用了字段collection，集合这个概念只有在分布式部署情况，索引被分布式放在多个服务器的时候才有意义。单服务器下更容易理解core，分布式collection与core的区别在章节13中讨论。

Solr home是包裹着一个或多个core的目录，单核或者是多核的配置在配置文件solr.xml中。solr.xml主要是针对于分布式配置，对于目前的示例程序来说，你可以忽略它。Solr也提供了增删查改core的API以方便编程，在12章中将探讨这些API。

当前你需要理解的是每个Solr服务器只有且仅有一个solr home 目录。Java的系统配置solr.solr.home设定了Solr 的home目录。图2.4显示了对于这个示例Solr服务器的缺省Solr home目录。

**图2.4 Solr示例的缺省Solr home目录，包含了一个在solr.xml中配置命名为collection1的core，collection1目录对应着名为collection1的core，包含了core-specific文件、Lucene 索引以及transaction日志。**

![图 2.4](https://raw.githubusercontent.com/yyjtry/solr_in_action/master/solr_chapter_2/2.4.PNG)

在第4章中我们会想学习Solr的主要配置文件（solrconfig.xml），schema.xml用来配置索引结构、文本分析，章节5将会详细描述。图2.4展示就是一个Solr home目录的基本结构。

为了方便探索Solr的高级功能，示例目录下还有两个Solr home 目录。其中example/example-DIH目录提供了一个core来了解Solr的DIH特性，example/multicore目录则展示了多核的配置。这些之后会继续探讨，现在继续学习示例，首先向索引中加入文件，在章节2.2中需要用到这些文件。

---

### 2.1.4. 为示例文档建立索引

刚开始使用Solr的时候，你的索引里没有文件，现在还只是个空服务器等着你填搜索数据。在第5章中将深入学习索引，现在大概过一遍，填些数据，开始尝试查询。开一个新的命令行，输入如下：

    cd $SOLR_INSTALL/example/exampledocs
    java -jar post.jar *.xml
    
之后你会应该会看到如下输出：

    SimplePostTool version 1.5
    Posting files to base url http://localhost:8983/solr/update using content-
    type application/xml..
    POSTing file gb18030-example.xml
    POSTing file hd.xml
    POSTing file ipod_other.xml
    POSTing file ipod_video.xml
    POSTing file manufacturers.xml
    POSTing file mem.xml
    POSTing file money.xml
    POSTing file monitor.xml
    POSTing file monitor2.xml
    POSTing file mp500.xml
    POSTing file sd500.xml
    POSTing file solr.xml
    POSTing file utf8-example.xml
    POSTing file vidcard.xml
    14 files indexed.
    COMMITting Solr index changes to http://localhost:8983/solr/update..

post.jar文件会调用HTTP POST向Solr发送XML文件，所有的文件发送到Solr之后，post.jar会处理一个commit，使得这些示例文件能被Solr找到。在Solr管理员界面执行查询全部（\*：\*），验证post是否成功。先在左边的下拉框中选定collection1，到达查询界面，图2.5显示了成功查询后的界面。

**图2.5 Solr管理界面的执行查询后的快照，以确认示例文档是否成功上传并被处理**
![图 2.5](https://raw.githubusercontent.com/yyjtry/solr_in_action/master/solr_chapter_2/2.5.PNG)

至此我们已经有了一个Solr示例，其上已经加载了一些示例文档。



















    





