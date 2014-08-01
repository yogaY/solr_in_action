# Solr_in_Action 2.4

标签（空格分隔）： solr_in_action 翻译

[TOC]

---

## 2.4. 修改example适应自己的需求

建立自己的服务器配置，两种选择：（1）直接在example目录下修改配置（2）拷贝example，重命名。推荐第二种

若选择了第二种方式，选定你的新名字比如`realestate/`，则新建步骤如下：
> 1. 创建`example/`的深拷贝，`cp -R realestate`
> 2. 清理拷贝的目录文件，删除无用的solr home目录，比如·example-DIH`，`multi-core`
> 3. 在你的solr home 目录下，重命名`collection1`为合适的名字`your_name`
> 4. 在`core.properties`中替换`collection1`为`your_name`，指向你的core，比如：`name = realestate`

注意，现在你还不需要更改Solr 的配置文件（比如solrconfig.xml，schema.xml），这些配置主要用来提升solr的使用，一步一步来，别想一口吃个肉包子。

> **扩展——清空索引**
> 有些时候你可能想要全部重来。停掉Solr之后，你可以删掉你的core的`data/`目录下的所有东西，之后重启，你的索引就清空了。

按照章节2.1.2的步骤重启solr。
你还好奇如何设定JVM、配置的备份、监控、Solr作为服务开启，这些对于生产环境很重要，在章节12中将对此讨论。






