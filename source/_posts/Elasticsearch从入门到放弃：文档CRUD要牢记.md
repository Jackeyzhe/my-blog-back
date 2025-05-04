---
title: Elasticsearch从入门到放弃：文档CRUD要牢记
date: 2019-11-24 21:16:46
tags: Elasticsearch笔记
---

在Elasticsearch中，文档（document）是所有可搜索数据的最小单位。它被序列化成JSON存储在Elasticsearch中。每个文档都会有一个唯一ID，这个ID你可以自己指定或者交给Elasticsearch自动生成。<!-- more -->

如果延续我们之前不恰当的对比RDMS的话，我认为文档可以类比成关系型数据库中的表。

### 元数据

前面我们提到，每个文档都有一个唯一ID来标识，获取文档时，“_id”字段记录的就是文档的唯一ID，它是元数据之一。当然，文档还有一些其他的元数据，下面我们来一一介绍

- _index：文档所属的索引名
- _type：文档所属的type
- _id：文档的唯一ID

有了这三个，我们就可以唯一确定一个document了，当然，7.0版本以后我们已经不需要_type了。接下来我们再来看看其他的一些元数据

- _source：文档的原始JSON数据
- _field_names：该字段用于索引文档中值不为null的字段名，主要用于exists请求查找指定字段是否为空
- _ignore：这个字段用于索引和存储文档中每个由于异常（开启了ignore_malformed）而被忽略的字段的名称
- _meta：该字段用于存储一些自定义的元数据信息
- _routing：用来指定数据落在哪个分片上，默认值是Id
- _version：文档的版本信息
- _score：相关性打分

### 创建文档

创建文档有以下4种方法：

- PUT /\<index\>/\_doc/<_id>
- POST /\<index\>/_doc/
- PUT /\<index>/\_create/<_id>
- POST /\<index>/\_create/<_id>

这四种方法的区别是，如果不指定id，则Elasticsearch会自动生成一个id。如果使用\_create的方法，则必须保证文档不存在，而使用\_doc方法的话，既可以创建新的文档，也可以更新已存在的文档。

在创建文档时，还可以选择一些参数。

#### 请求参数

- **if_seq_no**：当文档的序列号是指定值时才更新
- **if_primary_term**：当文档的primary term是指定值时才更新
- **op_type**：如果设置为create则指定id的文档必须不存在，否则操作失败。有效值为index或create，默认为index
- **op_type**：指定预处理的管道id
- **refresh**：如果设置为true，则立即刷新受影响的分片。如果是wait_for，则会等到刷新分片后，此次操作才对搜索可见。如果是false，则不会刷新分片。默认值为false
- **routing**：指定路由到的主分片
- **timeout**：指定响应时间，默认是30秒
- **master_timeout**：连接主节点的响应时长，默认是30秒
- **version**：显式的指定版本号
- **version_type**：指定版本号类型：internal、 external、external_gte、force
- **wait_for_active_shards**：处理操作之前，必须保持活跃的分片副本数量，可以设置为all或者任意正整数。默认是1，即只需要主分片活跃。

#### 响应包体

- **_shards**：提供分片的信息
- **_shards.total**：创建了文档的总分片数量
- **_shards.successful**：成功创建文档分片的数量
- **_shards.failed**：创建文档失败的分片数量
- **_index**：文档所属索引
- **_type**：文档所属type，目前只支持\_doc
- **_id**：文档的id
- **_version**：文档的版本号
- **_seq_no**：文档的序列号
- **_primary_term**：文档的主要术语
- **result**：索引的结果，created或者updated

我们在创建文档时，如果指定的索引不存在，则ES会自动为我们创建索引。这一操作是可以通过设置中的action.auto_create_index字段来控制的，默认是true。你可以修改这个字段，实现指定某些索引可以自动创建或者所有索引都不能自动创建的目的。

### 更新文档

了解了如何创建文档之后，我们再来看看应该如何更新一个已经存在的文档。其实在创建文档时我们就提到过，使用PUT /\<index>/\_doc/\<id>的方法就可以更新一个已存在的文档。除此之外，我们还有另一种更新文档的方法：

POST /\<index>/\_update/\<_id>

这两种更新有所不同。\_doc方法是先删除原有的文档，再创建新的。而\_update方法则是增量更新，它的更新过程是先检索到文档，然后运行指定脚本，最后重新索引。

还有一个区别就是\_update方法支持使用脚本更新，默认的语言是painless，你可以通过参数lang来进行设置。在请求参数方面，\_update相较于\_doc多了以下几个参数：

- **lang**：指定脚本语言
- **retry_on_conflict**：发生冲突时重试次数，默认是0
- **_source**：设置为false，则不返回任何检索字段
- **_source_excludes**：指定要从检索结果排除的source字段
- **_source_includes**：指定要返回的检索source字段

下面的一个例子是用脚本来更新文档

``` bash
curl -X POST "localhost:9200/test/_update/1?pretty" -H 'Content-Type: application/json' -d'
{
    "script" : {
        "source": "ctx._source.counter += params.count",
        "lang": "painless",
        "params" : {
            "count" : 4
        }
    }
}
'
```

#### Upsert

``` bash
curl -X POST "localhost:9200/test/_update/1?pretty" -H 'Content-Type: application/json' -d'
{
    "script" : {
        "source": "ctx._source.counter += params.count",
        "lang": "painless",
        "params" : {
            "count" : 4
        }
    },
    "upsert" : {
        "counter" : 1
    }
}
'
```

当指定的文档不存在时，可以使用upsert参数，创建一个新的文档，而当指定的文档存在时，该请求会执行script中的脚本。如果不想使用脚本，而只想新增/更新文档的话，可以使用doc_as_upsert。

``` bash
curl -X POST "localhost:9200/test/_update/1?pretty" -H 'Content-Type: application/json' -d'
{
    "doc" : {
        "name" : "new_name"
    },
    "doc_as_upsert" : true
}
'
```

#### update by query

这个API是用于批量更新检索出的文档的，具体可以通过一个例子来了解。

``` bash
curl -X POST "localhost:9200/twitter/_update_by_query?pretty" -H 'Content-Type: application/json' -d'
{
  "script": {
    "source": "ctx._source.likes++",
    "lang": "painless"
  },
  "query": {
    "term": {
      "user": "kimchy"
    }
  }
}
'
```

### 获取文档

ES获取文档用的是GET API，请求的格式是：

GET /\<index>/\_doc/<\_id>

它会返回文档的数据和一些元数据，如果你只想要文档的内容而不需要元数据时，可以使用

GET /\<index>/\_source/<\_id>

#### 请求参数

获取文档的有几个请求参数之前已经提到过，这里不再赘述，它们分别是：

- **refresh**
- **routing**
- **_source**
- **_source_excludes**
- **_source_includes**
- **version**
- **version_type**

而还有一些之前没提到过的参数，我们来具体看一下

- **preference**：用来 指定执行请求的node或shard，如果设置为\_local，则会优先在本地的分片执行
- **realtime**：如果设置为true，则请求是实时的而不是近实时。默认是true
- **stored_fields**：返回指定的字段中，store为true的字段

#### mget

mget是批量获取的方法之一，请求的格式有两种：

- GET /\_mget
- GET /\<index>/\_mget

第一种是在请求体中写index。第二种是把index放到url中，不过这种方式可能会触发ES的安全检查。

mget的请求参数和get相同，只是需要在请求体中指定doc的相关检索条件

**request**

``` bash
GET /_mget
{
    "docs" : [
        {
            "_index" : "jackey",
            "_id" : "1"
        },
        {
            "_index" : "jackey",
            "_id" : "2"
        }
    ]
}
```

**response**

``` bash
{
  "docs" : [
    {
      "_index" : "jackey",
      "_type" : "_doc",
      "_id" : "1",
      "_version" : 5,
      "_seq_no" : 6,
      "_primary_term" : 1,
      "found" : true,
      "_source" : {
        "user" : "ja",
        "tool" : "ES",
        "message" : "qwer"
      }
    },
    {
      "_index" : "jackey",
      "_type" : "_doc",
      "_id" : "2",
      "_version" : 1,
      "_seq_no" : 2,
      "_primary_term" : 1,
      "found" : true,
      "_source" : {
        "user" : "zhe",
        "post_date" : "2019-11-15T14:12:12",
        "message" : "learning Elasticsearch"
      }
    }
  ]
}
```

### 删除文档

CURD操作只剩下最后一个D了，下面我们就一起来看看ES中如何删除一个文档。

删除指定id使用的请求是

DELETE /\<index>/\_doc/<\_id>

在并发量比较大的情况下，我们在删除时通常会指定版本，以确定删除的文档是我们真正想要删除的文档。删除请求的参数我们在之前也都介绍过，想要具体了解的同学可以直接查看[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/docs-delete.html)。

#### delete by query

类似于update，delete也有一个delete by query的API。

POST /\<index>/\_delete_by_query

它也是要先按照条件来查询匹配的文档，然后删除这些文档。在执行查询之前，Elasticsearch会先为指定索引做一个快照，如果在执行删除过程中，要索引发生改变，则会导致操作冲突，同时返回删除失败。

如果删除的文档比较多，也可以使这个请求异步执行，只需要设置wait_for_completion=false即可。

这个API的refresh与delete API的refresh参数有所不同，delete中的refresh参数是设置操作是否立即可见，即只刷新一个分片，而这个API中的refresh参数则是需要刷新受影响的所有分片。

### Bulk API

最后，我们再来介绍一种特殊的API，批量操作的API。它支持两种写法，可以将索引名写到url中，也可以写到请求体中。

- POST /\_bulk

- POST /\<index>/_bulk

在这个请求中，你可以任意使用之前的CRUD请求的组合。

``` bash
curl -X POST "localhost:9200/_bulk?pretty" -H 'Content-Type: application/json' -d'
{ "index" : { "_index" : "test", "_id" : "1" } }
{ "field1" : "value1" }
{ "delete" : { "_index" : "test", "_id" : "2" } }
{ "create" : { "_index" : "test", "_id" : "3" } }
{ "field1" : "value3" }
{ "update" : {"_id" : "1", "_index" : "test"} }
{ "doc" : {"field2" : "value2"} }
'
```

请求体中使用的语法是newline delimited JSON（NDJSON）。具体怎么用呢？其实我们在上面的例子中已经有所展现了，对于index或create这样的请求，如果请求本身是有包体的，那么用换行符来表示下面的内容与子请求分隔，即为包体的开始。

例如上面例子中的index请求，它的包体就是{ "field1" : "value1" }，所以它会在index请求的下一行出现。

对于批量执行操作来说，单条操作失败并不会影响其他操作，而最终每条操作的结果也都会返回。

上面的例子执行完之后，我们得到的结果应该是

``` bash
{
   "took": 30,
   "errors": false,
   "items": [
      {
         "index": {
            "_index": "test",
            "_type": "_doc",
            "_id": "1",
            "_version": 1,
            "result": "created",
            "_shards": {
               "total": 2,
               "successful": 1,
               "failed": 0
            },
            "status": 201,
            "_seq_no" : 0,
            "_primary_term": 1
         }
      },
      {
         "delete": {
            "_index": "test",
            "_type": "_doc",
            "_id": "2",
            "_version": 1,
            "result": "not_found",
            "_shards": {
               "total": 2,
               "successful": 1,
               "failed": 0
            },
            "status": 404,
            "_seq_no" : 1,
            "_primary_term" : 2
         }
      },
      {
         "create": {
            "_index": "test",
            "_type": "_doc",
            "_id": "3",
            "_version": 1,
            "result": "created",
            "_shards": {
               "total": 2,
               "successful": 1,
               "failed": 0
            },
            "status": 201,
            "_seq_no" : 2,
            "_primary_term" : 3
         }
      },
      {
         "update": {
            "_index": "test",
            "_type": "_doc",
            "_id": "1",
            "_version": 2,
            "result": "updated",
            "_shards": {
                "total": 2,
                "successful": 1,
                "failed": 0
            },
            "status": 200,
            "_seq_no" : 3,
            "_primary_term" : 4
         }
      }
   ]
}
```

批量操作的执行过程相比多次单个操作而言，在性能上会有一定的提升。但同时也会有一定的风险，所以我们在使用的时候要非常的谨慎。

### 总结

本文我们先介绍了文档的基本概念和文档的元数据。接着又介绍了文档的CRUD操作和Bulk API。相信看完文章你对Elasticsearch的文档也会有一定的了解。那最后就请你启动你的Elasticsearch，然后亲自动手试一试这些操作，看看各种请求的参数究竟有什么作用。相信亲手实验过一遍之后你会对这些API有更深的印象。