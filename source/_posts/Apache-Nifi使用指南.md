---
title: Apache Nifi使用指南
top: false
toc: true
mathjax: true
date: 2020-01-09 12:49:41
index_img: https://i.loli.net/2020/01/09/6E4IuLwCiAm15KW.png
summary:
tags:
 - docker
 - apache nifi
 - etl
categories: etl
---

## 简介
Apache NiFi是什么？NiFi官网给出如下解释：“一个易用、强大、可靠的数据处理与分发系统”。通俗的来说，即Apache NiFi 是一个易于使用、功能强大而且可靠的数据处理和分发系统，其为数据流设计，它支持高度可配置的指示图的数据路由、转换和系统中介逻辑。

## 架构

单节点架构

![NiFi Architecture Diagram](https://nifi.apache.org/docs/nifi-docs/html/images/zero-master-node.png)

集群架构图

![NiFi Cluster Architecture Diagram](https://nifi.apache.org/docs/nifi-docs/html/images/zero-master-cluster.png)

**web-sever**

其目的在于提供基于HTTP的命令和控制API。

**Flow Controller**

这是操作的核心，以Processor为处理单元，提供了用于运行的扩展线程，并管理扩展接收资源时的调度。

**Extensions**

在其他文档中描述了各种类型的NiFi扩展，Extensions的关键在于扩展在JVM中操作和执行。

**FlowFile Repository**

FlowFile库的作用是NiFi跟踪记录当前在流中处于活动状态的给定流文件的状态，其实现是可插拔的，默认的方法是位于指定磁盘分区上的一个持久的写前日志。FlowFile库的作用是NiFi跟踪记录当前在流中处于活动状态的给定流文件的状态，其实现是可插拔的，默认的方法是位于指定磁盘分区上的一个持久的写前日志。

**Content Repository**

Content库的作用是给定流文件的实际内容字节所在的位置，其实现也是可插拔的。默认的方法是一种相对简单的机制，即在文件系统中存储数据块。

**Provenance Repository**

Provenance库是所有源数据存储的地方，支持可插拔。默认实现是使用一个或多个物理磁盘卷，在每个位置事件数据都是索引和可搜索的。

## docker部署

nifi需要通过集群实现多租户，在standalone模式下，可以通过docker启动多个实例来实现多用户通过不同端口访问各自的nifi

```shell
docker run --name nifi01 \
  --restart=always \
  -p 8090:8080 \
  -p 10000:10000 \
  -v /data/nifi:/data/nifi/ \
  -d \
  apache/nifi:latest

  docker run --name nifi02 \
  --restart=always \
  -p 8094:8080 \
  -p 10001:10000 \
  -v /data/nifi02:/data/nifi02/ \
  -d \
  apache/nifi:latest
```

启动nifi，停止nifi

```shell
docker start nifi
docker stop nifi
```

## NiFi Processor

Flow Controller是NiFi的核心，Flow Controller扮演者文件交流的处理器角色，维持着多个处理器的连接并管理各个Processer，Processor则是实际处理单元。

![1572501106736](assets/processor.png)

Processor包含各种类型的组件，如amazon、attributes、hadoop等，可通过前缀进行轻易辨识，如Get、Fetch开头代表获取，如getFile、getFTP、FetchHDFS，execute代表执行，如ExecuteSQL、ExecuteProcess、ExecuteFlumeSink等均可较容易知其简单用途。右边选择hadoop则会显示所有hadoop相关的processor，如图所示

![1572501279158](assets/hadoop.png)

与hadoop相关的processor有读写HDFS文件，写Hbase，读写parquet文件等

## Nifi实战Demo

### 通过Nifi实现把指定文件夹中的文件移动到另一个文件夹

- step 1选取processor

  选取getfile及putfile的process，并连接

 [![geifile.png](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/scaled-840-0/geifile.png)](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/geifile.png)

[![flow.png](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/scaled-840-0/flow.png)](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/flow.png)

- step 2 填写getfile及putfile相关属性

  双击getfile或者putfile的processor，property中填写源数据文件夹及目标文件夹，其他参数按需进行配置，此demo按照默认参数填写，对于putfile processor中的setting栏，设置勾选自动终止策略，当putfile失败或者成功的时候就停止此processor

  [![properties.png](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/scaled-840-0/properties.png)](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/properties.png)

[![putfile.png](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/scaled-840-0/putfile.png)](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/putfile.png)

[![putfile_config.png](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/scaled-840-0/putfile_config.png)](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/putfile_config.png)

- step 3 启动

  空白处右键选择start，状态栏显示绿色代表processor在执行，假如/data/nifi/input有文件此时开启工作流后/data/nifi/input对应也会有该文件

[![start.png](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/scaled-840-0/start.png)](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/start.png)

### 数据库表和表同步  
表到表的同步,NIFI默认sql查询出来的数据为Avro格式,所以需要先将Avro格式转化为json格式,再将json转换为sql语句,最后使用PUTSQL处理器将数据存入数据库。需要用到ExecuteorSQL、ConvertAvroToJSON、ConvertJSONToSQL、PUTSQL四种处理器。
[![table2table.png](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/scaled-840-0/table2table.png)](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/table2table.png) 

- step 1 使用ExecuteSQL配置数据源  
从Components ToolBar上将processor拖拽到画布上，选择ExecuteSQL处理器，右键或双击该处理器编辑properties。该处理器原生提供三种数据库连接池，提供了大多数数据库驱动，另外还可以自定义连接池的名称。  
[![ExecuteSQL.png](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/scaled-840-0/ExecuteSQL.png)](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/ExecuteSQL.png)
  - Database Connection Pooling Service 提供数据库连接池服务  
  - postgresql驱动程序[下载](https://jdbc.postgresql.org/download.html)  
 [![driverConfig.png](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/scaled-840-0/driverConfig.png)](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/driverConfig.png)
  - 驱动配置完成后点击⚡,使能后查看是否报错  
- step 2 使用ConvertAvroToJSON将Avro格式转换为json格式  
[![ConvertAvroToJSON.png](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/scaled-840-0/ConvertAvroToJSON.png)](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/ConvertAvroToJSON.png)
- step 3 使用ConvertJSONToSQL将json数据转换为SQL语句  
[![ConvertJSONToSQL.png](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/scaled-840-0/ConvertJSONToSQL.png)](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/ConvertJSONToSQL.png) 
- step 4 使用PUTSQL将数据存入到数据库，配置完成后数据流启动后可以看到数据流动过程，再到数据库中验证结果。  
[![PUTSQL.png](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/scaled-840-0/PUTSQL.png)](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/PUTSQL.png)
**Tips:基本案例跑通后可以在Operate栏createTemplate,可以在web页面直接导出xml格式的数据处理流程重复使用减少配置时间。**  


## Trouble Shooting

- 原始文件夹或者目标文件夹没有权限，process右上角会显示报错信息，此时应该将文件夹设置对应权限，将拥有者改为nifi用户，此命令应该用root权限执行，若nifi在docker中，应该到使用root用户登录到对应container执行

```
linux版本
sudo chown -R nifi:nifi /data/nifi/input
sudo chown -R nifi:nifi /data/nifi/output
docker版本
docker exec -it -u root nifi /bin/bash
chown -R nifi:nifi /data/nifi/input
chown -R nifi:nifi /data/nifi/output
```

报错信息：

[![error.png](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/scaled-840-0/error.png)](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/error.png)

- putfile或者getfile的properties填写错误的时候，对应processor上面会显示感叹号，按照提示修改即可，如getfile中源文件夹为/data/nifi/input01，系统中没有此文件夹，则会显示对应提示信息

[![error01.png](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/scaled-840-0/error01.png)](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/error01.png)