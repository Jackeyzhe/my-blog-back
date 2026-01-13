---
title: Flink源码阅读：Kafka Connector
date: 2026-01-12 14:39:21
tags: Flink
---

本文我们来梳理 Kafka Connector 相关的源码。<!-- more -->

### 自定义 Source 和 Sink

在介绍 Kafka Connector 之前，我们先来看一下在 Flink 中是如何支持自定义 Source 和 Sink 的。我们来看一张 Flink 官方文档提供的图。

![table-connector](https://res.cloudinary.com/dxydgihag/image/upload/v1768288302/Blog/flink/22/table_connectors.png)

这张图展示了 Connector 的基本体系结构，三层架构也非常清晰。

#### Metadata

首先是最上层的 MetaData，CREATE TABLE 会更新 Catalog，然后被转换为 TableAPI 的 CatalogTable，CatalogTable 实例用于表示动态表（Source 或 Sink 表）的元信息。

#### Planning

在解析和优化程序时，会将 CatalogTable 转换为 DynamicTableSource 和 DynamicTableSink，分别用于查询和插入数据，这两个实例的创建都需要对应的工厂类，工厂类的完整路径需要放到这个配置文件中。

```bash
META-INF/services/org.apache.flink.table.factories.Factory
```

如果有需要的话，我们还可以在解析过程中配置编码和解码方法。

在 Source 端，通过三个接口支持不同的查询能力。

- ScanTableSource：用于消费 changelog 流，扫描的数据支持 insert、updata、delete 三种类型。ScanTableSource 还支持很多其他的功能， 都是通过接口提供的。具体可以看参考这个连接
  
  ```bash
  https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/dev/table/sourcessinks/#source-abilities
  ```

- LookupTableSource：LookupTableSource 不会全量读取表的数据，它在需要时会发送请求，懒加载数据。目前只支持 insert-only 变更模式。

- VectorSearchTableSource：使用一个输入向量来搜索数据，并返回最相似的 Top-K 行数据。

在 Sink 端，通过 DynamicTableSink 来实现具体的写入逻辑，这里也提供了一些用于扩展能力的接口。具体参考

```bash
https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/dev/table/sourcessinks/#sink-abilities
```

#### Runtime

逻辑解析完成后，会到 Runtime 层。这里就是定义几个 Provider，在 Provider 中实现和连接器具体的交互逻辑。

#### 小结

当我们需要创建一个自定义的 Source 和 Sink 时，就可以通过以下步骤实现。

1. 定义 Flink SQL 的 DDL，需要定义相应的 Options。

2. 实现 DynamicTableSourceFactory 和 DynamicTableSinkFactory，并把实现类的具体路径写到配置文件中。

3. 实现 DynamicTableSource 和 DynamicTableSink，这里需要处理 SQL 层的元数据。

4. 提供 Provider，将逻辑层与底层 DataStream 关联起来。

5. 编写底层算子，实现 Source 和 Sink 接口。

### Kafka Connector 的实现

带着这些知识，我们一起来看一下 Kafka Connector 相关的源码。

Kafka Connector 代码目前已经是一个独立的项目了。项目地址是

```bash
https://github.com/apache/flink-connector-kafka
```

#### Factory

我们首先找到定义的工厂类

```java
org.apache.flink.streaming.connectors.kafka.table.KafkaDynamicTableFactory
org.apache.flink.streaming.connectors.kafka.table.UpsertKafkaDynamicTableFactory
```

以 KafkaDynamicTableFactory 为例，它同时实现了 DynamicTableSourceFactory 和 DynamicTableSinkFactory 两个接口。

KafkaDynamicTableFactory 包含以下几个方法。

![KafkaDynamicTableFactory](https://res.cloudinary.com/dxydgihag/image/upload/v1768294001/Blog/flink/22/KafkaDynamicTableFactory.png)

- factoryIdentifier：返回一个唯一标识符，对应 Flink SQL 中 connector='xxx' 这个配置。

- requiredOptions：必填配置集合。

- optionalOptions：选填配置集合。

- forwardOptions：直接传递到 Runtime 层的配置集合。

- createDynamicTableSource：创建 DynamicTableSource。

- createDynamicTableSink：创建 DynamicTableSink。

#### Source端

工厂类的 createDynamicTableSource 方法创建了 DynamicTableSource，我们来看一下创建的逻辑。

```java
public DynamicTableSource createDynamicTableSource(Context context) {
    final TableFactoryHelper helper = FactoryUtil.createTableFactoryHelper(this, context);

    final Optional<DecodingFormat<DeserializationSchema<RowData>>> keyDecodingFormat =
            getKeyDecodingFormat(helper);

    final DecodingFormat<DeserializationSchema<RowData>> valueDecodingFormat =
            getValueDecodingFormat(helper);

    helper.validateExcept(PROPERTIES_PREFIX);

    final ReadableConfig tableOptions = helper.getOptions();

    validateTableSourceOptions(tableOptions);

    validatePKConstraints(
            context.getObjectIdentifier(),
            context.getPrimaryKeyIndexes(),
            context.getCatalogTable().getOptions(),
            valueDecodingFormat);

    final StartupOptions startupOptions = getStartupOptions(tableOptions);

    final BoundedOptions boundedOptions = getBoundedOptions(tableOptions);

    final Properties properties = getKafkaProperties(context.getCatalogTable().getOptions());

    // add topic-partition discovery
    final Duration partitionDiscoveryInterval =
            tableOptions.get(SCAN_TOPIC_PARTITION_DISCOVERY);
    properties.setProperty(
            KafkaSourceOptions.PARTITION_DISCOVERY_INTERVAL_MS.key(),
            Long.toString(partitionDiscoveryInterval.toMillis()));

    final DataType physicalDataType = context.getPhysicalRowDataType();

    final int[] keyProjection = createKeyFormatProjection(tableOptions, physicalDataType);

    final int[] valueProjection = createValueFormatProjection(tableOptions, physicalDataType);

    final String keyPrefix = tableOptions.getOptional(KEY_FIELDS_PREFIX).orElse(null);

    final Integer parallelism = tableOptions.getOptional(SCAN_PARALLELISM).orElse(null);

    return createKafkaTableSource(
            physicalDataType,
            keyDecodingFormat.orElse(null),
            valueDecodingFormat,
            keyProjection,
            valueProjection,
            keyPrefix,
            getTopics(tableOptions),
            getTopicPattern(tableOptions),
            properties,
            startupOptions.startupMode,
            startupOptions.specificOffsets,
            startupOptions.startupTimestampMillis,
            boundedOptions.boundedMode,
            boundedOptions.specificOffsets,
            boundedOptions.boundedTimestampMillis,
            context.getObjectIdentifier().asSummaryString(),
            parallelism);
}
```

在这个方法中，首先要获取到 key 和 value 的解码格式。接着是各种参数校验和获取必要的属性。最后创建 KafkaDynamicSource 实例。

获取解码格式需要用到 DeserializationFormatFactory 工厂，DeserializationFormatFactory 有多个实现类，对应了多种格式的反序列化方法。

![DeserializationFormatFactory](https://res.cloudinary.com/dxydgihag/image/upload/v1768297428/Blog/flink/22/DeserializationFormatFactory.png)

我们来看比较常见的 Json 格式的工厂 JsonFormatFactory。

```java
public DecodingFormat<DeserializationSchema<RowData>> createDecodingFormat(
        DynamicTableFactory.Context context, ReadableConfig formatOptions) {
    FactoryUtil.validateFactoryOptions(this, formatOptions);
    JsonFormatOptionsUtil.validateDecodingFormatOptions(formatOptions);

    final boolean failOnMissingField = formatOptions.get(FAIL_ON_MISSING_FIELD);
    final boolean ignoreParseErrors = formatOptions.get(IGNORE_PARSE_ERRORS);
    final boolean jsonParserEnabled = formatOptions.get(DECODE_JSON_PARSER_ENABLED);
    TimestampFormat timestampOption = JsonFormatOptionsUtil.getTimestampFormat(formatOptions);

    return new ProjectableDecodingFormat<DeserializationSchema<RowData>>() {
        @Override
        public DeserializationSchema<RowData> createRuntimeDecoder(
                DynamicTableSource.Context context,
                DataType physicalDataType,
                int[][] projections) {
            final DataType producedDataType =
                    Projection.of(projections).project(physicalDataType);
            final RowType rowType = (RowType) producedDataType.getLogicalType();
            final TypeInformation<RowData> rowDataTypeInfo =
                    context.createTypeInformation(producedDataType);
            if (jsonParserEnabled) {
                return new JsonParserRowDataDeserializationSchema(
                        rowType,
                        rowDataTypeInfo,
                        failOnMissingField,
                        ignoreParseErrors,
                        timestampOption,
                        toProjectedNames(
                                (RowType) physicalDataType.getLogicalType(), projections));
            } else {
                return new JsonRowDataDeserializationSchema(
                        rowType,
                        rowDataTypeInfo,
                        failOnMissingField,
                        ignoreParseErrors,
                        timestampOption);
            }
        }

        @Override
        public ChangelogMode getChangelogMode() {
            return ChangelogMode.insertOnly();
        }

        @Override
        public boolean supportsNestedProjection() {
            return jsonParserEnabled;
        }
    };
}
```

在创建解码格式时，最重要的是创建运行时的解码器，也就是 DeserializationSchema，在 JsonFormatFactory 中，有 JsonParserRowDataDeserializationSchema 和 JsonRowDataDeserializationSchema 两种实现，分别是用于将 JsonParser 和 JsonNode 转换成为 RowData，具体的逻辑都在 createNotNullConverter 方法中。

了解完解码格式后，我们把视角拉回到 KafkaDynamicSource，它实现了三个接口 ScanTableSource、SupportsReadingMetadata、SupportsWatermarkPushDown。分别用于消费数据，读取元数据和生成水印。

```java
public ScanRuntimeProvider getScanRuntimeProvider(ScanContext context) {
    final DeserializationSchema<RowData> keyDeserialization =
            createDeserialization(context, keyDecodingFormat, keyProjection, keyPrefix);

    final DeserializationSchema<RowData> valueDeserialization =
            createDeserialization(context, valueDecodingFormat, valueProjection, null);

    final TypeInformation<RowData> producedTypeInfo =
            context.createTypeInformation(producedDataType);

    final KafkaSource<RowData> kafkaSource =
            createKafkaSource(keyDeserialization, valueDeserialization, producedTypeInfo);

    return new DataStreamScanProvider() {
        @Override
        public DataStream<RowData> produceDataStream(
                ProviderContext providerContext, StreamExecutionEnvironment execEnv) {
            if (watermarkStrategy == null) {
                watermarkStrategy = WatermarkStrategy.noWatermarks();
            }
            DataStreamSource<RowData> sourceStream =
                    execEnv.fromSource(
                            kafkaSource, watermarkStrategy, "KafkaSource-" + tableIdentifier);
            providerContext.generateUid(KAFKA_TRANSFORMATION).ifPresent(sourceStream::uid);
            return sourceStream;
        }

        @Override
        public boolean isBounded() {
            return kafkaSource.getBoundedness() == Boundedness.BOUNDED;
        }

        @Override
        public Optional<Integer> getParallelism() {
            return Optional.ofNullable(parallelism);
        }
    };
}
```

在 ScanRuntimeProvider 的逻辑中，先获取到反序列化器，也就是刚刚我们提到的 DeserializationSchema。然后创建 KafkaSource 实例，它是 Source 的实现类，也就是执行引擎层了。
