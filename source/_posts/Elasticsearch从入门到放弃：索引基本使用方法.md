---
title: Elasticsearch从入门到放弃：索引基本使用方法
date: 2019-10-13 01:07:09
tags: Elasticsearch笔记
---

前文我们提到，Elasticsearch的数据都存储在索引中，也就是说，索引相当于是MySQL中的数据库。是最基础的概念。今天分享的也是关于索引的一些常用的操作。<!-- more -->

### 创建索引

```bash
curl -X PUT "localhost:9200/jackey?pretty"
```

ES创建索引使用PUT请求即可，上面是最简单的新建一个索引的方法，除此之外，你还可以指定：

- Settings
- Mappings
- aliases

索引名称有以下限制：

1. 必须是小写
2. 不能包含：`\`,`/`,`*`, `?`, `"`, `<`, `>`, `|`, ` `(空格),`,`, `#`
3. 在ES7.0以前索引名可以包含冒号，但是7.0之后不支持了
4. 不能以`-`,`_`和`+`开头
5. 不能是`.`或`..`
6. 长度不能超过255字节

请求支持的一些参数有：

- **wait_for_active_shards**：继续操作前，必须处于active状态的分片数，默认是1，也可以设置为all或者不大于总分片数的任意正整数
- **timeout**：设置等待响应的超时时间，默认是30秒
- **master_timeout**：连接master节点响应的超时时间，默认是30秒

前面我们提到创建索引时可以指定三种属性，这三种属性都需要放在body中。

#### aliases

索引的别名，一个别名可以赋给多个索引。

给一个index起别名的方式有两种，一种是创建index时候在body中增加aliases，另一种是通过更新已有索引的方式增加。

方式一：

```bash
curl -X PUT "localhost:9200/jackey?pretty" -H 'Content-Type: application/json' -d'
{
    "aliases" : {
        "alias_1" : {},
        "alias_2" : {
            "filter" : {
                "term" : {"user" : "kimchy" }
            },
            "routing" : "kimchy"
        }
    }
}
'
```

方式二：

```bash
curl -X POST "localhost:9200/_aliases?pretty" -H 'Content-Type: application/json' -d'
{
    "actions" : [
        { "add" : { "index" : "jackey", "alias" : "alias1" } }
    ]
}
'
```

方式一中，我们还在body中增加了filter和routing。这主要是用于指定使用别名的条件。指定了filter后，通过alias_2，只能访问user为kimchy的document。而routing的值被用来路由，即alias_2只能路由到指定的分片。此外还有index_routing和search_routing，它们和routing类似，这里不做过多解释了。还有一个比较重要的属性是is_write_index，这个属性默认是false，如果设置成true，表示可以通过这个别名来写索引，默认情况下，别名像一个软链接，是不可以修改原索引的。

此外，还可以使用通配符为多个索引增加相同的别名

```bash
curl -X POST "localhost:9200/_aliases?pretty" -H 'Content-Type: application/json' -d'
{
    "actions" : [
        { "add" : { "index" : "test*", "alias" : "all_test_indices" } }
    ]
}
'
```

除了add，还可以使用remove来删除别名

```bash
curl -X POST "localhost:9200/_aliases?pretty" -H 'Content-Type: application/json' -d'
{
    "actions" : [
        { "remove" : { "index" : "test1", "alias" : "alias1" } }
    ]
}
'
```

#### Settings

先看一个例子：

```bash
curl -X PUT "localhost:9200/twitter?pretty" -H 'Content-Type: application/json' -d'
{
    "settings" : {
        "index" : {
            "number_of_shards" : 3, 
            "number_of_replicas" : 2 
        }
    }
}
'
```

索引的setting分为静态和动态两种。静态的只能在索引创建或关闭时设置；动态的则可以使用update-index-settings API来实时设置。上面的例子中，number_of_shards属于静态设置，number_of_replicas属于动态设置。

索引可以设置的setting可以在官方文档的[Index modules](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/index-modules.html)查看，下面我会挑几个我认为比较重要的介绍一下。

先从静态开始：

- **index.number_of_shards**：指定索引的分片数，只能在创建索引时设置。默认是1，最大可以设置为1024。这是出于安全考虑的一种保护措施。最大值可以通过设置系统变量来控制export ES_JAVA_OPTS="-Des.index.max_number_of_shards=128"
- **index.routing_partition_size**：可以路由的分片数量，同样只能在创建索引时指定，默认值为1.这个值必须小于number_of_shards（除非number_of_shards的值也是1）

动态setting：

- **index.number_of_replicas**：每个分片的副本数，默认是1
- **index.auto_expand_replicas**：基于数据节点可以自动扩展的副本数，默认为为false。可以设置为一个区间，以短线分隔，例如「0-5」，也可以设置成all。需要注意的是，副本的自动扩展并不会考虑其他的分配规则。这有可能导致集群状态变成黄色
- **index.search.idle.after**：分片被认为搜索空闲之前没有收到请求或搜索的时间。默认30秒。
- **index.refresh_interval**：刷新操作的执行频率，默认是1s。如果设置成-1，表示不会刷新。如果没有显式设置，分片在收到搜索请求前至少index.search.idle.after秒内不会后台刷新
- **index.max_result_window**：返回结果的最大数量，默认是10000（一万）。搜索返回结果占用的内存和时间受到这个值的限制
- **index.routing.rebalance.enable**：是否允许分片的自平衡。默认是all，允许所有分片重新平衡。还可以设置为primaries，只允许主分片重新平衡。replicas只允许从分片重新平衡。none不允许分片重新平衡。

除了以上静态setting和动态setting之外，setting中还可以设置一些其他的值，例如分词器等，这些我们以后再做更详细的介绍。

#### Mappings

```bash
curl -X PUT "localhost:9200/test?pretty" -H 'Content-Type: application/json' -d'
{
    "settings" : {
        "number_of_shards" : 1
    },
    "mappings" : {
        "properties" : {
            "field1" : { "type" : "text" }
        }
    }
}
'
```

Mapping主要用于帮助Elasticsearch理解每个域中数据的类型。7.0.0之前mapping的定义通常包括type名称。Elasticsearch支持的数据类型比较多，其中比较核心的简单数据类型包括：

- 字符串: text和keyword
- 整数 : byte, short, integer, long
- 浮点数: float, double
- 布尔型: boolean
- 日期: date

其他的类型，我们以后会做更加详细的介绍。

### 删除索引

删除索引使用的是DELETE请求。

```bash
curl -X DELETE "localhost:9200/jackey?pretty"
```

你可以在路径中指定具体索引，也可以使用通配符，需要删除多个索引时，可以使用逗号分隔。如果要删除全部索引，可以指定索引为_all或*（不要这么做）。在生产环境，我们通过在elasticsearch.yml文件中将action.destructive_requires_name配置为true来禁止这些危险的操作。

删除操作支持的参数有以下几种：

- **allow_no_indices**：如果设置为true，则通配符或_all匹配不到索引时不会报错
- **expand_wildcards**：控制通配符可以扩展到的索引类型。all：可以扩展到所有的索引。open：只能扩展到打开的索引。closed：只能扩展到关闭的索引。none：不接受通配符表达式。默认是open
- **ignore_unavailable**：如果设置为true，不存在或关闭的索引不会在返回中。默认是false
- **timeout**：指定等待返回响应的最长时间。默认是30秒
- **master_timeout**：连接master节点响应的超时时间，默认是30秒

### 打开/关闭索引

前面我们已经提到过了打开/关闭索引。被关闭的索引几乎不能对它进行任何操作，它只是用来保留数据的。而打开或关闭索引通常需要重启分片来使操作生效。具体的操作如下：

```bash
curl -X POST "localhost:9200/jackey/_open?pretty"
```

```bash
curl -X POST "localhost:9200/jackey/_close?pretty"
```

支持的参数有：

- **allow_no_indices**
- **expand_wildcards**
- **ignore_unavailable**
- **wait_for_active_shards**
- **timeout**
- **master_timeout**

这些参数在前面都有介绍。这里就不再赘述了。

### 拆分索引

随着数据的越来越多，我们可能会有拆分索引的需求，感谢ES为我们提供了便利。

```bash
curl -X POST "localhost:9200/twitter/_split/split-twitter-index?pretty" -H 'Content-Type: application/json' -d'
{
  "settings": {
    "index.number_of_shards": 2
  }
}
'
```

在拆分索引之前，要保证索引是只读状态，并且集群健康状态为green。设置只读的方法是：

```bash
curl -X PUT "localhost:9200/my_source_index/_settings?pretty" -H 'Content-Type: application/json' -d'
{
  "settings": {
    "index.blocks.write": true 
  }
}
'
```

拆分索引的具体操作是：

1. 创建一个和源索引相同的目标索引，主分片要大于源索引
2. 建立从源索引到目标索引的硬连接
3. 创建低级索引后，再对document做Hash操作。这是为了删除属于不同分片的document
4. 恢复目标索引，就像重新打开关闭的索引一样

### 总结

关于索引的使用就先介绍到这里。还有很多不完善的地方，以后会继续补充。想要了解更多详细信息的同学可以查看[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/indices.html)。

