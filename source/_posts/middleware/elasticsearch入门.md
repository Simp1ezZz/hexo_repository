# Elasticsearch入门：索引、文档与搜索

## 前言

Elasticsearch（简称ES）是一个基于Lucene的分布式搜索和分析引擎，被广泛应用于全文检索、日志分析、安全分析等场景。凭借其强大的搜索能力和水平扩展特性，ES成为ELK（Elasticsearch、Logstash、Kibana）栈的核心组件。本文将介绍Elasticsearch的核心概念、索引管理、文档操作和搜索查询。

## Elasticsearch架构

### 基本概念对比

```
┌─────────────────────────────────────────────────────────────────┐
│              Elasticsearch vs 传统数据库                         │
├─────────────────────┬───────────────────────────────────────────┤
│   Elasticsearch     │         传统数据库                          │
├─────────────────────┼───────────────────────────────────────────┤
│  Index (索引)       │         Database (数据库)                   │
│  Type (类型)        │         Table (表)                          │
│  Document (文档)    │         Row (行)                            │
│  Field (字段)       │         Column (列)                        │
│  Mapping (映射)     │         Schema (表结构)                      │
│  Shard (分片)       │         -                                  │
│  Replica (副本)     │         -                                  │
└─────────────────────┴───────────────────────────────────────────┘
```

### 集群架构

```
┌─────────────────────────────────────────────────────────────────┐
│                     Elasticsearch集群                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Node 1 (Master + Data)  │  Node 2 (Data)  │  Node 3 (Data)    │
│  ┌──────────────────┐   │  ┌───────────┐  │  ┌───────────┐    │
│  │ Primary Shard 0   │   │  │ Shard 1   │  │  │ Shard 2   │    │
│  │ Replica Shard 1   │   │  │ Replica 0 │  │  │ Replica 0 │    │
│  └──────────────────┘   │  └───────────┘  │  └───────────┘    │
│                          │                 │                    │
└──────────────────────────┴─────────────────┴────────────────────┘
                              │
                              ▼
                    ┌─────────────────────┐
                    │    Kibana (可视化)    │
                    └─────────────────────┘
```

## 索引操作

### 创建索引

```bash
# 创建索引（设置分片和副本）
PUT /my-index
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "analysis": {
      "analyzer": {
        "my_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "stop"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "ik_max_word",
        "fields": {
          "keyword": {
            "type": "keyword"
          }
        }
      },
      "content": {
        "type": "text",
        "analyzer": "ik_max_word"
      },
      "author": {
        "type": "keyword"
      },
      "publish_date": {
        "type": "date"
      },
      "views": {
        "type": "long"
      },
      "status": {
        "type": "keyword"
      },
      "tags": {
        "type": "keyword"
      }
    }
  }
}
```

### 字段类型

```json
// 字符串类型
"text": {           // 分词全文检索
  "type": "text",
  "analyzer": "standard"
}
"keyword": {        // 精确值，不分词
  "type": "keyword"
}

// 数值类型
"integer": { "type": "integer" }
"long": { "type": "long" }
"float": { "type": "float" }
"double": { "type": "double" }
"short": { "type": "short" }
"byte": { "type": "byte" }

// 日期类型
"date": {
  "type": "date",
  "format": "yyyy-MM-dd||yyyy-MM-dd HH:mm:ss||epoch_millis"
}

// 布尔类型
"is_deleted": { "type": "boolean" }

// 地理位置
"location": { "type": "geo_point" }

// 对象类型
"user": {
  "type": "object",
  "properties": {
    "name": { "type": "keyword" },
    "age": { "type": "integer" }
  }
}

// 嵌套类型
"comments": {
  "type": "nested",
  "properties": {
    "user": { "type": "keyword" },
    "content": { "type": "text" }
  }
}
```

### 索引管理

```bash
# 查看索引列表
GET /_cat/indices?v

# 查看索引结构
GET /my-index/_mapping

# 查看索引设置
GET /my-index/_settings

# 删除索引
DELETE /my-index

# 打开/关闭索引
POST /my-index/_close
POST /my-index/_open

# 索引模板
PUT /_template/logs-template
{
  "index_patterns": ["logs-*"],
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {
      "@timestamp": { "type": "date" },
      "message": { "type": "text" }
    }
  }
}
```

## 文档操作

### 添加文档

```bash
# 添加文档（指定ID）
PUT /my-index/_doc/1
{
  "title": "Elasticsearch入门",
  "content": "Elasticsearch是一个强大的搜索引擎...",
  "author": "张三",
  "publish_date": "2024-01-15",
  "views": 1000,
  "status": "published",
  "tags": ["elasticsearch", "搜索", "入门"]
}

# 添加文档（自动生成ID）
POST /my-index/_doc
{
  "title": "ES高级特性",
  "content": "本文介绍ES的高级特性...",
  "author": "李四",
  "publish_date": "2024-01-20",
  "views": 500,
  "tags": ["elasticsearch", "高级"]
}

# 批量添加
POST /_bulk
{ "index": { "_index": "my-index", "_id": "2" } }
{ "title": "批量操作", "content": "Bulk API示例..." }
{ "index": { "_index": "my-index", "_id": "3" } }
{ "title": "聚合分析", "content": "ES聚合查询..." }
```

### 查询文档

```bash
# 根据ID查询
GET /my-index/_doc/1

# 查询所有文档
GET /my-index/_search

# 查询所有（简化）
GET /my-index/_doc/_search
{
  "query": { "match_all": {} }
}

# 分页查询
GET /my-index/_search
{
  "from": 0,
  "size": 10,
  "query": {
    "match_all": {}
  },
  "sort": [
    { "publish_date": "desc" },
    { "_score": "desc" }
  ]
}
```

### 更新文档

```bash
# 更新文档（完整替换）
PUT /my-index/_doc/1
{
  "title": "Elasticsearch入门（更新版）",
  "content": "更新后的内容...",
  "author": "张三",
  "publish_date": "2024-01-15",
  "views": 1500,
  "status": "published",
  "tags": ["elasticsearch", "搜索"]
}

# 部分更新
POST /my-index/_update/1
{
  "doc": {
    "views": 2000,
    "status": "archived"
  }
}

# 使用脚本更新
POST /my-index/_update/1
{
  "script": {
    "source": "ctx._source.views += params.count",
    "params": {
      "count": 100
    }
  }
}

# 批量更新
POST /_bulk
{ "update": { "_index": "my-index", "_id": "1" } }
{ "doc": { "views": 3000 } }
{ "update": { "_index": "my-index", "_id": "2" } }
{ "doc": { "views": 1500 } }
```

### 删除文档

```bash
# 删除文档
DELETE /my-index/_doc/1

# 删除查询匹配的文档
POST /my-index/_delete_by_query
{
  "query": {
    "term": {
      "status": "deleted"
    }
  }
}

# 清空索引
DELETE /my-index/_doc/_query
{
  "query": {
    "match_all": {}
  }
}
```

## 搜索查询

### 全文查询

```bash
# match查询（分词匹配）
GET /my-index/_search
{
  "query": {
    "match": {
      "title": "Elasticsearch 入门"
    }
  }
}

# match_phrase查询（短语匹配）
GET /my-index/_search
{
  "query": {
    "match_phrase": {
      "title": {
        "query": "Elasticsearch 入门",
        "slop": 1
      }
    }
  }
}

# multi_match查询（多字段匹配）
GET /my-index/_search
{
  "query": {
    "multi_match": {
      "query": "搜索 引擎",
      "fields": ["title^2", "content"],
      "type": "best_fields",
      "tie_breaker": 0.3
    }
  }
}

# query_string查询
GET /my-index/_search
{
  "query": {
    "query_string": {
      "default_field": "content",
      "query": "(elasticsearch OR solr) AND 入门",
      "default_operator": "AND"
    }
  }
}
```

### 精确查询

```bash
# term查询（精确值，不分词）
GET /my-index/_search
{
  "query": {
    "term": {
      "author": "张三"
    }
  }
}

# terms查询（多值精确匹配）
GET /my-index/_search
{
  "query": {
    "terms": {
      "tags": ["elasticsearch", "搜索"]
    }
  }
}

# range查询（范围）
GET /my-index/_search
{
  "query": {
    "range": {
      "views": {
        "gte": 100,
        "lte": 1000
      }
    }
  }
}

# exists查询（存在字段）
GET /my-index/_search
{
  "query": {
    "exists": {
      "field": "author"
    }
  }
}

# missing查询（字段不存在）- ES 5.x后已移除，用bool must_not exists代替
GET /my-index/_search
{
  "query": {
    "bool": {
      "must_not": [
        { "exists": { "field": "author" } }
      ]
    }
  }
}
```

### 布尔查询

```bash
# bool查询
GET /my-index/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": "elasticsearch" } }
      ],
      "should": [
        { "match": { "content": "搜索" } },
        { "match": { "content": "lucene" } }
      ],
      "must_not": [
        { "term": { "status": "deleted" } }
      ],
      "filter": [
        { "range": { "views": { "gte": 100 } } }
      ]
    }
  }
}
```

### 嵌套查询

```bash
# nested查询（嵌套文档）
PUT /articles
{
  "mappings": {
    "properties": {
      "title": { "type": "text" },
      "comments": {
        "type": "nested",
        "properties": {
          "user": { "type": "keyword" },
          "content": { "type": "text" },
          "date": { "type": "date" }
        }
      }
    }
  }
}

GET /articles/_search
{
  "query": {
    "nested": {
      "path": "comments",
      "query": {
        "bool": {
          "must": [
            { "match": { "comments.user": "张三" } },
            { "match": { "comments.content": "很好" } }
          ]
        }
      }
    }
  }
}
```

### 地理查询

```bash
# 地理位置
PUT /locations
{
  "mappings": {
    "properties": {
      "name": { "type": "keyword" },
      "location": { "type": "geo_point" }
    }
  }
}

# 添加带地理位置的文档
PUT /locations/_doc/1
{
  "name": "天安门",
  "location": {
    "lat": 39.907,
    "lon": 116.391
  }
}

# 附近查询
GET /locations/_search
{
  "query": {
    "geo_distance": {
      "distance": "10km",
      "location": {
        "lat": 39.9,
        "lon": 116.4
      }
    }
  }
}
```

## 聚合分析

### 聚合函数

```bash
# 统计聚合
GET /my-index/_search
{
  "size": 0,
  "aggs": {
    "total_views": {
      "sum": { "field": "views" }
    },
    "avg_views": {
      "avg": { "field": "views" }
    },
    "max_views": {
      "max": { "field": "views" }
    },
    "min_views": {
      "min": { "field": "views" }
    },
    "view_stats": {
      "stats": { "field": "views" }
    }
  }
}

# 分桶聚合
GET /my-index/_search
{
  "size": 0,
  "aggs": {
    "by_author": {
      "terms": {
        "field": "author",
        "size": 10
      },
      "aggs": {
        "total_views": {
          "sum": { "field": "views" }
        }
      }
    }
  }
}

# 日期直方图聚合
GET /my-index/_search
{
  "size": 0,
  "aggs": {
    "by_month": {
      "date_histogram": {
        "field": "publish_date",
        "calendar_interval": "month",
        "format": "yyyy-MM"
      }
    }
  }
}
```

## Java客户端

### Spring Data Elasticsearch

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
</dependency>
```

```java
// 实体类
@Document(indexName = "my-index")
@Data
public class Article {
    @Id
    private String id;

    @Field(type = FieldType.Text, analyzer = "standard")
    private String title;

    @Field(type = FieldType.Text, analyzer = "standard")
    private String content;

    @Field(type = FieldType.Keyword)
    private String author;

    @Field(type = FieldType.Date, format = DateFormat.date)
    private LocalDate publishDate;

    @Field(type = FieldType.Long)
    private Long views;

    @Field(type = FieldType.Keyword)
    private List<String> tags;
}

// Repository
public interface ArticleRepository extends ElasticsearchRepository<Article, String> {
    List<Article> findByTitle(String title);
    List<Article> findByAuthor(String author);
    List<Article> findByViewsGreaterThanEqual(Long views);
}

// Service
@Service
@RequiredArgsConstructor
public class ArticleService {
    private final ArticleRepository repository;
    private final ElasticsearchTemplate template;

    public void save(Article article) {
        repository.save(article);
    }

    public Page<Article> search(String keyword, Pageable pageable) {
        Query query = new NativeQueryBuilder()
            .withQuery(q -> q
                .multiMatch(m -> m
                    .query(keyword)
                    .fields("title^2", "content")
                )
            )
            .withPageable(pageable)
            .build();

        return template.queryForPage(query, Article.class);
    }
}
```

## 常见面试题

**Q1：Elasticsearch和关系型数据库的区别？**

> 参考答案：ES是面向文档的NoSQL数据库，以JSON格式存储数据。关系型数据库有Database->Table->Row的层级，而ES是Index->Type->Document（7.x后Type被移除）。ES擅长全文检索和复杂查询，但不适合强事务场景；关系型数据库适合需要ACID特性的场景。ES的倒排索引使其在搜索性能上远超传统数据库。

**Q2：ES是如何实现全文检索的？**

> 参考答案：ES基于Lucene，每个字段都会建立倒排索引。全文检索时，将搜索词分词后，通过倒排索引快速找到包含这些词的文档。相比正排索引（文档->词），倒排索引（词->文档）大大提升了搜索效率。ES还支持分词器定制、相关性计算（TF/IDF/BM25）、高亮显示等功能。

**Q3：ES如何保证高可用？**

> 参考答案：ES通过分片（Shard）和副本（Replica）实现高可用。每个索引可以设置多个分片，每个分片可以有多个副本。主分片负责读写，副本分片提供数据冗余和读能力。当某个节点宕机，ES会自动将副本升级为主分片，实现故障转移。集群会选举Master节点管理分片分配。

**Q4：ES的搜索性能优化有哪些方法？**

> 参考答案：1）合理设计分片数，避免过多小分片或过少大分片；2）使用filter替代query进行精确过滤（filter不计算评分）；3）开启路由（routing）直接定位到目标分片；4）合理使用doc_values和fielddata；5）数据预热；6）使用异步搜索处理大结果集；7）配置慢日志排查慢查询。

## 总结

```
┌────────────────────────────────────────────────────────────────┐
│                    Elasticsearch核心要点                       │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  核心概念：                                                       │
│  ├─ Index: 索引（类似数据库）                                     │
│  ├─ Document: 文档（类似行）                                      │
│  ├─ Mapping: 映射（类似表结构）                                   │
│  ├─ Shard: 分片（数据分片）                                      │
│  └─ Replica: 副本（数据冗余）                                    │
│                                                                │
│  搜索能力：                                                       │
│  ├─ 全文检索: match, multi_match, query_string                   │
│  ├─ 精确匹配: term, terms, range                                │
│  ├─ 布尔逻辑: bool (must/should/must_not/filter)                │
│  ├─ 嵌套查询: nested                                            │
│  └─ 聚合分析: sum/avg/max/min/terms/date_histogram              │
│                                                                │
│  使用场景：                                                       │
│  ├─ 全文搜索引擎                                                 │
│  ├─ 日志分析（ELK）                                             │
│  ├─ 安全分析                                                     │
│  ├─ 应用性能监控                                                │
│  └─ 地理位置搜索                                                │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

Elasticsearch是现代搜索和日志分析架构中不可或缺的组件。掌握其核心概念和查询语法，能够帮助我们构建高效的搜索系统和数据分析平台。
