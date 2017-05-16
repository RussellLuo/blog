categories:
- 技术

tags:
- Elasticsearch

title: Elasticsearch Analyzer 浅析
---

## 基本概念

Elasticsearch Analyzer 由三部分构成：（零个或多个）character filters、（一个 ）tokenizers、（零个或多个）token filters。

Analyzer 主要用于两个地方：

1. 索引文档时，分析处理「文档字段」analyzed fields
2. 搜索文档时，分析处理「查询字符串」query strings

Analyzer 中各个部分的工作顺序如下：

(Input)  -->  [Character Filters]  -->  [Tokenizers]  -->  [Token filters]  --> (Tokens or Terms)

更多说明参考 [Analysis][1] 和 [Mapping -> Mapping parameters -> analyzer][2]。

另外，[Elasticsearch Analyzer 的内部机制][3] 这篇文章总结得也很到位。

## 内置 Analyzers

ElasticSearch 包括多种内置的 Analyzers。更多说明参考 [Analysis -> Analyzers][4]。

例如，如果要使用内置的「标准 Analyzer」，则需要指定 `type` 为 `standard`。


## 自定义 Analyzers

ElasticSearch 也支持自定义 Analyzers，这正是其强大之处。自定义 Analyzer 必须指定 `type` 为 `custom`。

例如：

```
$ curl -XDELETE 'http://localhost:9200/test'

$ curl -XPUT 'http://localhost:9200/test' -d '
{
  "index": {
    "analysis": {
      "analyzer": {
        "my_analyzer": {
          "type": "custom",
          "tokenizer": "uax_url_email",
          "filter": ["lowercase"],
          "char_filter": ["html_strip"]
        }
      }
    }
  }
}'

$ curl -XGET 'http://localhost:9200/test/_analyze' -d '
{
  "analyzer": "my_analyzer",
  "text": "this is a test url <b>http://www.example.com</b>"
}'
```

如果只是想测试 Analyzer 是否工作，也可以不用指定索引。更多说明参考 [Indices APIs -> Analyze][5]。

例如上述 Analyzer 可以这样测试：

```
$ curl -XGET 'http://localhost:9200/_analyze' -d '
{
  "tokenizer": "uax_url_email",
  "filter": ["lowercase"],
  "char_filter": ["html_strip"],
  "text": "this is a test url <b>http://www.example.com</b>"
}'
```


[1]: https://www.elastic.co/guide/en/elasticsearch/reference/2.3/analysis.html
[2]: https://www.elastic.co/guide/en/elasticsearch/reference/2.3/analyzer.html
[3]: http://mednoter.com/all-about-analyzer-part-one.html
[4]: https://www.elastic.co/guide/en/elasticsearch/reference/2.3/analysis-analyzers.html
[5]: https://www.elastic.co/guide/en/elasticsearch/reference/2.3/indices-analyze.html
