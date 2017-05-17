categories:
- 技术

tags:
- Elasticsearch

title: Elasticsearch 基于时间的索引
---

## 应用场景

对于数据量较大的业务功能（比如日志），如果使用单个 ES 索引来存储文档，与日俱增的数据量很快就会使得单个索引过大，因为无法水平扩展，最终会导致机器空间不足。这种大数据量的场景下，需要对数据进行切分，将数据分段存储在不同的索引中。

[Sizing Elasticsearch][1] 介绍了常用的几种数据切分方法，因为这两天在工作中刚好用到过，所以在这里重点总结下 “基于时间的索引” ([time-based indices][2]) 的管理技巧。

## 选择时间范围

根据数据增长速度的不同，可以选择按天索引（索引名称形如 2017-05-16），或者按月索引（索引名称形如 2017-05）等等。

## 设计索引模板

面对这么多不断新增的索引，如何管理它们的 settings 和 mappings 呢？一个一个地去手动维护，无疑是个噩梦。这时，就需要用到 ES 的 [Index Templates][3] 机制。

Index Templates 的基本原理是：首先预定义一个或多个 “索引模板”（index template，其中包括 settings 和 mappings 配置）；然后在创建索引时，一旦索引名称匹配了某个 “索引模板”，ES 就会自动将该 “索引模板” 包含的配置（settings 和 mappings）应用到这个新创建的索引上面。

以日志为例，假设我们的 ES 索引需求如下：

1. 按天索引（索引名称形如 log-2017-05-16）
2. 每天的日志数据，只会进入当天的索引
3. 搜索的时候，希望搜索范围是所有的索引（借助 alias）

基于上述索引需求，对应的 “索引模板” 可以设计为：

```bash
$ curl -XPUT http://localhost:9200/_template/log_template -d '{
  "template": "log-*",
  "settings": {
    "number_of_shards": 1
  },
  "mappings": {
    "log": {
      "dynamic": false,
      "properties": {
        "content": {
          "type": "string"
        },
        "created_at": {
          "type": "date",
          "format": "dateOptionalTime"
        }
      }
    }
  },
  "aliases": {
    "search-logs": {}
  }
}'
```

两点说明：

1. 创建索引时，如果索引名称的格式形如 "log-*"，ES 会自动将上述 settings 和 mappings 应用到该索引
2. aliases 的配置，告诉 ES 在每次创建索引时，自动为该索引添加一个名为 "search-logs" 的 alias（别名）

## 索引与搜索

基于上述 “索引模板” 的设计，索引与搜索的策略就很直接了。

索引策略：每天的数据，只索引到当天对应的索引。比如，2017 年 5 月 16 日这天的数据，只索引到 log-2017-05-16 这个索引当中。

搜索策略：因为搜索需求是希望全量搜索，所以在搜索的时候，索引名称使用 "search-logs" 这个 alias 即可。


更多关于 “如何有效管理基于时间的索引” 的技巧，可以参考 [Managing Elasticsearch time-based indices efficiently][4]


[1]: https://www.elastic.co/blog/found-sizing-elasticsearch
[2]: https://www.elastic.co/guide/en/elasticsearch/guide/current/time-based.html
[3]: https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-templates.html
[4]: https://www.elastic.co/blog/managing-time-based-indices-efficiently
