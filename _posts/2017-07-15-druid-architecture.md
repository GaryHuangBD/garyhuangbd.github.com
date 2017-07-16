---
layout: post
title:  "Druid原理(-)--基本概念"
date:   2017-07-16 18:18:00
categories: Druid
tags: Druid
---

* content
{:toc}

鉴于很多小伙伴都想了解Druid的原理，以及具体它能做那些事情，以及怎么做这些事情，我决定写一个系列关于Druid的文章。本篇文章先大概描述Druid的一些基础概念，后续在针对每一块深入，也欢迎小伙伴们积极评论给一些建议，后续文章会根据意见情况和问题集中点展开。  

Druid首先是一个BI的OLAP查询系统，它最开始被设计的目的是为了支持即席查询实时和历史数据。Druid可以实时接入数据，支持多种方式接入数据、多种数据格式的解析，支持PB级的数据查询，节点可水平扩展，支持冷热数据分离，支持LAMDA架构离线批量Update数据，可用于灵活的数据探索。  







## Druid数据结构  

传统的RDBMS中只有维度的概念，分析数据时我们可以聚合维度和筛选维度。而Druid的数据是不同的，拿下面这份数据来举例(数据来源于Druid官方)：  

| timestamp | publisher | advertiser | gender | country | click | price |
|-----------|-----------|------------|--------|---------|-------|-------|
| 2011-01-01T01:01:35Z | bieberfever.com | google.com | Male | USA | 0 | 0.65 |
| 2011-01-01T01:03:63Z | bieberfever.com | google.com | Male | USA | 0 | 0.62 |
| 2011-01-01T01:04:51Z | bieberfever.com | google.com | Male | USA | 1 | 0.45 |
| 2011-01-01T01:00:00Z | ultratrimfast.com | google.com | Female | UK | 0 | 0.87 |
| 2011-01-01T02:00:00Z | ultratrimfast.com | google.com | Female | UK | 0 | 0.99 |
| 2011-01-01T02:00:00Z | ultratrimfast.com | google.com | Female | UK | 1 | 1.53 | 

Druid可以根据分析的需求来建立模型，不需要的信息可以不存储，有点类似于建立数据集市的概念。假如目的是分析publisher和advertiser的点击量，建立模型包括以下三部分内容：  
* **时间列** 每一条数据都必须有此列，此列会决定数据所在的Segment(后面会介绍)，所有的查询也都需要此列，在上面对应的数据列为`timestamp`，在系统中会存储的字段名为`__time`；  
* **维度列** Druid会针对这些列建立bitmap的倒排索引，用于筛选数据，结合时间列便能快速找到需要查询的数据。根据假设的分析模型，我们只需要`publisher`和`advertiser`，而其他例如`gender`和`country`并非我们需要分析的列，便可以在建立模型的时候忽略，需要注意的是Druid只有String型的维度列，如果我们需要对维度列做数字范围的筛选，需要做特殊处理  
* **聚合列(metric column)** Druid会根据我们定义的时间粒度，再结合维度列在数据接入的时候对数据进行聚合，并根据聚合类型，用不同的数据结构来保存这一列。这一列算是Druid比较复杂的一个概念，与下面介绍的预聚合有很大关系，稍后我会进一步解释。在假设的分析模型中，我们需要对`click`列求`sum`。  

于是我们可以建立一个简单的分析模型：  

```
"dataSchema": {
    "dataSource": "wiki",
    "parser": {
      "type": "string",       ＃说明读取的每一行数据都是都是string类型
      "parseSpec": {
        "format": "json",     ＃说明数据的格式需要通过json来解析
        "timestampSpec": {    ＃指明时间列字段和格式
          "column": "timestamp",
          "format": "auto"
        },
        "dimensionsSpec": {   ＃指明需要索引的维度，只有加到此处的维度才可以查询
          "dimensions": ["publisher", "advertiser"]
        }
      }
    },
    "metricsSpec": [
    {
       "name": "click_sum",
       "fieldName": "click",
       "type": "longSum"
    }],
    "granularitySpec" : {     #预聚合相关的配置
      "segmentGranularity" : "HOUR",
      "queryGranularity" : "HOUR"
    }
  }
```

>上面的分析模型是一个简单的使用，Druid本身的配置性非常强，我会在后续章节介绍这些配置的实现原理。

## 预聚合  

上面的样例数据我们可以称之为原始数据，为我们直接采集到的数据，每天这样的数据有千亿甚至万亿。基于这些原始数据，如果我们需要分析点击量，成本是非常高的，因为我们尽管我们可以通过维度筛选出一部分数据，但是数据量还是很巨大，需要扫描这些数据来聚合，如此便会直接导致查询时长加长。为了避免这种情况，Druid采用了预聚合的设计，在数据接入的时候，就开始根据定义的维度，对数据进行聚合。Druid的预聚合分为两个级别，第一级是时间粒度，第二级是维度。  

### 时间粒度  

例子中模型的配置有`segmentGranularity` 和`queryGranularity`两个关键的参数，可以理解为一个是段的粒度，一个是查询的最小粒度。首先，Druid会根据`timestamp`和`segmentGranularity`来决定数据所属的段；然后，再根据`timestamp`和`queryGranularity`来决定存储在Druid的`__time`。  

### 维度  

Druid的预聚合过程也是其索引的过程，通常Druid会在内存中使用ConcurrentSkipListMap存储数据，以TimeAndDims(时间和维度值信息)作为key，根据配置的`metricsSpec`对数据进行聚合，聚合的结果作为value。为了避免OOM，最后会以bitmapIndex的数据格式周期性的落地到磁盘存储。  

于是，样例数据最后存储的格式大概为：  

 Segment `sampleData_2011-01-01T01:00:00:00Z_2011-01-01T02:00:00:00Z_v1_0` 
|__time|publisher|advertiser|rows|click_sum|
|------|---------|----------|----|---------|
| 2011-01-01T01:00:00Z | ultratrimfast.com | google.com | 1 | 0 |
| 2011-01-01T01:00:00Z | bieberfever.com | google.com | 3 | 1 |


Segment `sampleData_2011-01-01T02:00:00:00Z_2011-01-01T03:00:00:00Z_v1_0` 
|__time|publisher|advertiser|rows|click_sum|
|------|---------|----------|----|---------|
| 2011-01-01T02:00:00Z | ultratrimfast.com | google.com | 2 | 1 |

>`segmentGranularity`为小时，所以对应的段都是小时级别。`queryGranularity`为小时，聚合的结果对应的__time也是整点，会将每小时的数据全部聚合到其所属小时的开始时间。另外说明，倒数第二例为总共的行数，最后一列为点击数。  

## Druid集群  

Druid集群按照功能分为不同节点类型，这些功能节点相互之间没有绝对依赖，可以独立工作，这在一定程度上保证了Druid的稳定性。我总结的Druid的结构如下图：  
![Druid架构图](http://img2.ph.126.net/trs2ZSa2XoqYSkysqKXrwA==/6632343199188865139.jpg)   

Druid主要的节点类型包括：  

* **Overlord** 管理Druid的索引任务，通过overlord可以启动和关闭Task，并可以获取当前Task的状态；  
* **Coordinator** 管理Druid的数据，通过该节点可以删除、重新加载数据段，并可以获取数据段的状态信息；  
* **Broker**  主要是查询数据，分发查询任务给Historical或者Task，并合并其返回的结果；    
* **Historical** 一旦索引成功的数据都会移交到该节点，尤其提供相应数据的查询；   
* **MiddleManager** 主要是为了启动Peon进程；  
* **Peon** 对数据进行索引，对应Druid的Task，不同的Task功能会有差异，对于实时Task，其本身还承担着查询自身接入的数据。  

>Druid还有一种特殊的节点RealtimeNode，但是考虑到其是长进程，不论稳定性、灵活性上考虑，还是从监控和重新调度来说，不如Peon方便，所以建议不再使用，而可以利用Peon进行索引即可。  

另外，Druid还依赖于三个外部组件：Zookeeper、分布式文件系统和数据库。Druid需要Zookeeper来同步各节点的状态信息，来支持整个系统的运行；需要一个分布式文件系统来作为DeepStorage，从而支持数据的备份，Druid已经支持HDFS/S3/DFS等分布式文件系统；还有需要数据库来存储Druid的元数据信息，如当前所有Segment信息、执行的Task信息，一些集群的配置信息等等。  

Druid集群索引的工作流程大致如下(不同的Task会有所差异)：  

1. 发起定制的Task给Overlord节点，Overlord根据调度策略来选择MiddleManager节点调度起香应的Task;  
2. MiddleManager监听到Task，则更根据Task信息启动相应的Peon进程，启动成功后，更改Task的状态；  
3. Peon进程正式执行Task，开始接入数据，并更改状态为Running；  
4. Task执行完成后，上传Segment到分布式文件系统；  
5. 上传的Segment信息写入数据库，Coordinator节点根据数据库里面获知到未分配的Segment信息，将其分配相应的Historical；  
6. Historical感知到相应的Segment，从分布式文件系统下载，通过ZK向集群声明提供该Segment的查询服务；  
7. Task任务完成。  






