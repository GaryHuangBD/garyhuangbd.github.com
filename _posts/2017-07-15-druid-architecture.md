---
layout: post
title:  "Druid的基本概念"
date:   2017-07-10 17:27:00
categories: Druid
tags: Druid
---

* content
{:toc}

鉴于很多小伙伴都想了解Druid的原理，以及具体它能做那些事情，以及怎么做这些事情，我决定写一个系列关于Druid的文章。本篇文章先大概描述Druid的一些基础概念，后续在针对每一块深入，也欢迎小伙伴们积极评论给一些建议，后续文章会根据意见情况和问题集中点展开。  

Druid首先是一个BI的OLAP查询系统，它最开始被设计的目的是为了支持即席查询实时和历史数据。Druid可以实时接入数据，支持多种方式接入数据、多种数据格式的解析，支持PB级的数据查询，节点可水平扩展，支持冷热数据分离，支持LAMDA架构离线批量Update数据，可用于灵活的数据探索。  







## Druid数据结构  

传统的RDBMS中，我们只有维度的概念，分析数据时我们可以聚合维度和筛选维度。而Druid的数据是不同的，拿下面这份数据来举例(数据来源于Druid官方)：  

    timestamp             publisher          advertiser  gender  country  click  price
    2011-01-01T01:01:35Z  bieberfever.com    google.com  Male    USA      0      0.65
    2011-01-01T01:03:63Z  bieberfever.com    google.com  Male    USA      0      0.62
    2011-01-01T01:04:51Z  bieberfever.com    google.com  Male    USA      1      0.45
    2011-01-01T01:00:00Z  ultratrimfast.com  google.com  Female  UK       0      0.87
    2011-01-01T02:00:00Z  ultratrimfast.com  google.com  Female  UK       0      0.99
    2011-01-01T02:00:00Z  ultratrimfast.com  google.com  Female  UK       1      1.53  

对于Druid来说，我们可以根据自己分析的需求来建立自己的模型，不需要的信息可以不存储，有点类似于建立数据集市的概念。假如我们的目的是分析publisher和advertiser的点击量，建立模型包括以下三部分内容：  
* **时间列** 每一条数据都必须有此列，此列会决定数据所在的Segment(后面会介绍)，所有的查询也都需要此列，在上面对应的数据列为`timestamp`，在系统中会存储的字段名为`__time`；  
* **维度列** Druid会针对这些列建立bitmap的倒排索引，用于筛选数据，结合时间列便能快速找到需要查询的数据。根据假设的分析模型，我们只需要`publisher`和`advertiser`，而其他例如`gender`和`country`并非我们需要分析的列，便可以在建立模型的时候忽略，需要注意的是Druid只有String型的维度列，如果我们需要对维度列做数字范围的筛选，需要做特殊处理  
* **聚合列(metric column)** Druid会根据我们定义的时间粒度，再结合维度列在数据接入的时候对数据进行聚合，并根据聚合类型，用不同的数据结构来保存这一列。这一列算是Druid比较复杂的一个概念，与下面介绍的预聚合有很大关系，稍后我会进一步解释。在假设的分析模型中，我们需要对`click`列求`sum`。  

于是我们可以建立一个简单的分析模型：  

```
"dataSchema": {
    "dataSource": "wiki",
    "parser": {
      "type": "string",
      "parseSpec": {
        "format": "json",
        "timestampSpec": {
          "column": "timestamp",
          "format": "auto"
        },
        "dimensionsSpec": {
          "dimensions": ["publisher", "advertiser"]
        }
      }
    },
    "metricsSpec": [
    {
       "name": "click_sum",
       "fieldName": "click",
       "type": "longSum"
    }]
  }
```

## 预聚合  








