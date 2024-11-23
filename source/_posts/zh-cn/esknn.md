---
title: es knn 最近邻搜索的实践
date: 2024年11月23日 22:08:55
updated:
keywords:
slug: elastic-knn
cover: /image/esknn.png
top_image:
comments: false
maincolor:
draft: true
categories:
  - ai 技术
tags:
  - elasticsearch
typora-root-url: ./esknn
description:
---

# 引言

最近公司在升级 ES 集群，从 v7.4 版本升级到 v8 版本，跨越一个大版本，除了业务上使用的 go 语言的 ES 开源库要替换之外，搜索的模块也因版本升级需要重新适配到业务上，正好就来整理一下 ES v8 版本的 Knn 搜索的相关知识

# Knn 是什么？

Knn ([K-nearest-neighbors](https://zh.wikipedia.org/wiki/K-%E8%BF%91%E9%82%BB%E7%AE%97%E6%B3%95))，k 近邻算法，用于计算特征空间中 K 个最接近的特征向量。细分多种算法，可以通过计算距离，计算距离的算法可以采用欧氏距离、余弦距离，或者是将其映射到一个图中，计算离搜索向量最近的 k 个节点。**其目的都是计算向量之间的距离，转换为通俗语义就是相似度，并找到理论上最相似的 K 个向量**

只需要将业务的数据转换为向量形式，就可以进行搜索。

它有什么用呢？

在人工智能领域，模型可以将输入的数据处理成特征，通过输入特征与已有的特征库的特征进行比较，就能找出最相似的 K 个特征，对用的是识别的领域。

同时也可以应用在根据用户喜好推荐相似产品、根据用户喜好、地理位置等信息推荐的附近的东西。

# 近似 Knn

近似 kNN 搜索方法可以帮助你快速找到最接近查询向量的 k 个向量。此搜索方法的逻辑与其他搜索方法的逻辑不同。近似 kNN 搜索方法对 Elasticsearch 集群的性能有特殊要求。你可以根据下方说明优化它：

1、Elasticsearch 将每个段（Segment）的数据存储在 分层可导航小世界（HNSW）图中，HNSW 图的构建在向量数据的索引过程中需要很长时间。建议你增加客户端的超时时间，并使用批量写入请求写入数据。

2、你可以减少索引的段（Segment）数量，也可以将索引的所有段合并为 1 个段，以提高搜索效率

3、你必须确保 Elasticsearch 集群中数据节点的内存空间大于所有向量数据和索引结构所占用的总空间

4、在近似 kNN 搜索期间，请勿向 Elasticsearch 集群写入大量数据或更新 Elasticsearch 集群中的数据。

HNSW 分层可导航小世界是一种图数据结构，它维护不同层中靠近的元素之间的链接，每层都包含相互链接的元素，并且还链接到其他层的元素，每层包含更多的元素，底层包含了所有的元素。

它的一个特点是搜索的过程中会跟踪一定量的候选节点（num_candidates），这个数量越大则搜索结果越精确，但是搜索速度会越慢。

因此，它允许用户通调控过候选节点数量，来平衡性能和搜索召回率。

## 建立索引

当你创建要在其中执行近似 kNN 搜索的索引时，请将 index 参数设置为 true，并在索引的映射中配置 similarity 参数。以下代码提供了一个配置示例：

```
PUT image-index
{
  "mappings": {
    "properties": {
      "image-vector": {
        "type": "dense_vector",
        "dims": 3,
        "index": true,
        "similarity": "l2_norm"
      },
      "title": {
        "type": "text"
      },
      "file-type": {
        "type": "keyword"
      }
    }
  }
}
```

字段解释：

1、type: 用于存储浮点数的字段的数据类型。将该参数设置为 dense_vector。

2、dims: 每个向量的维数。如果 index 为 true，则 dims 参数的值不能超过 1024。如果将 index 参数设置为 false，则 dims 参数的值不能超过 2048。

3、index: 指定是否为近似 kNN 搜索生成新索引。要执行近似 kNN 搜索，请将 index 参数设置为 true。默认值：false。

4、similarity:

- l2_norm 计算向量之间的欧氏距离。用于计算分数的公式： `1/(1 + l2_norm(query, vector)^2)` 。

- dot_product 计算两个向量的点积。分数的计算取决于 element_type 参数。
- cosine 计算向量之间的余弦相似度。使用余弦算法的最有效方法是将两个向量归一化为一个单位长度，以替换 dot_product 算法。用于计算分数的公式： `(1 + cosine(query, vector))/2` 。

## 写入数据

```
POST image-index/_bulk?refresh=true
{ "index": { "_id": "1" } }
{ "image-vector": [1, 5, -20], "title": "moose family", "file-type": "jpg" }
{ "index": { "_id": "2" } }
{ "image-vector": [42, 8, -15], "title": "alpine lake", "file-type": "png" }
{ "index": { "_id": "3" } }
{ "image-vector": [15, 11, 23], "title": "full moon", "file-type": "jpg" }
```

## 搜索

```
POST image-index/_search
{
  "knn": {
    "field": "image-vector",
    "query_vector": [-5, 9, -12],
    "k": 10,
    "num_candidates": 100
  },
  "fields": [ "title", "file-type" ]
}
```

字段解释：

1、field: 要搜索的向量字段的名称

2、query_vector： 查询向量。查询向量的维数必须与 field 参数指定的向量字段相同

3、k: 要返回的最近邻数。k 参数的值必须小于 num_candidates 参数的值

4、num_candidates： 每个分片需要搜索的最近邻候选项的数量。该值不能超过 10000

5、filter： 用于筛选文档的域特定语言 （DSL） 语句。近似 kNN 搜索返回与筛选条件匹配的前 k 个文档。如果不配置此参数，则匹配所有文档

# 精确 Knn

精确 kNN 用于与搜索结果集中每个的特征都进行距离计算，并将前 K 个相似的结果返回。

相比于近似 kNN，它的搜索性能差一些，但是它的索引写入速度会比近似 kNN 的索引快，因为近似 kNN 需要对写入的数据建立 HNSW 图。

**建议：索引数据量小于 100 万条的，使用精确 Knn 没什么影响，如果你有大量文档要检索，建议使用近似 knn 搜索，在牺牲一定精确度情况下，能显著提升性能**

以下是两者检索文档数量和性能的对比：

![](./knn_compare.png)

## 建立索引

```
PUT zl-index
{
  "mappings": {
    "properties": {
      "product-vector": {
        "type": "dense_vector",
        "dims": 5,
        "index": false
      },
      "price": {
        "type": "long"
      }
    }
  }
}
```

字段说明：

1、type: 用于存储浮点数的字段的数据类型。将该参数设置为 dense_vector

2、dim: 每个向量的维数

3、index: 指定是否为确切的 kNN 搜索生成新索引。默认值：false。对于精确 kNN 搜索，index 参数是可选的。您可以将此参数留空或将此参数设置为 false，以提高精确 kNN 搜索的效率

## 写入数据

```
POST zl-index/_bulk?refresh=true
{ "index": { "_id": "1" } }
{ "product-vector": [230.0, 300.33, -34.8988, 15.555, -200.0], "price": 1599 }
{ "index": { "_id": "2" } }
{ "product-vector": [-0.5, 100.0, -13.0, 14.8, -156.0], "price": 799 }
{ "index": { "_id": "3" } }
{ "product-vector": [0.5, 111.3, -13.0, 14.8, -156.0], "price": 1099 }
```

## 搜索

```
POST zl-index/_search
{
  "query": {
    "script_score": {
      "query" : {
        "bool" : {
          "filter" : {
            "range" : {
              "price" : {
                "gte": 1000
              }
            }
          }
        }
      },
      "script": {
        "source": "cosineSimilarity(params.queryVector, 'product-vector') + 1.0",
        "params": {
          "queryVector": [-0.5, 90.0, -10, 14.8, -156.0]
        }
      }
    }
  }
}
```

script_score 查询支持的向量函数有：

1、[cosineSimilarity](https://www.elastic.co/guide/en/elasticsearch/reference/8.6/query-dsl-script-score-query.html#vector-functions-cosine) 计算查询向量和文档向量之间的余弦相似度

2、[dotProduct](https://www.elastic.co/guide/en/elasticsearch/reference/8.6/query-dsl-script-score-query.html#vector-functions-dot-product) 计算查询向量和文档向量的点积

3、[l1norm](https://www.elastic.co/guide/en/elasticsearch/reference/8.6/query-dsl-script-score-query.html#vector-functions-l1) 计算查询向量和文档向量之间的 L1 距离（曼哈顿距离）

4、[l2norm](https://www.elastic.co/guide/en/elasticsearch/reference/8.6/query-dsl-script-score-query.html#vector-functions-l2) 计算查询向量和文档向量之间的 L2 距离（欧几里得距离）

# 总结

我们介绍了 KNN 搜索的用途，以及它的两种检索模式，我们解释了如何去选择，需要根据业务规模以及检索的结果精确度进行权衡。
