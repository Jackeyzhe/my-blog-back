---
title: 使用ModelMapper的一次踩坑经历
date: 2018-09-11 22:08:32
tags: Java
---

在实际项目中，我们常常需要把两个相似的对象相互转换，其目的是在对外提供数据时需要将一部分敏感数据（例如：密码、加密token等）隐藏起来。最普通的方法是，新建一个对象，将需要的值逐个set进去。如果有多组需要这样转换的对象，那么就需要做很多只是get/set这样无意义的工作。<!--more-->

在这样的背景下，ModelMapper诞生了，它是一个简单、高效、智能的对象映射工具。它的使用非常简单，首先添加maven依赖

``` java
<dependency>
  <groupId>org.modelmapper</groupId>
  <artifactId>modelmapper</artifactId>
  <version>2.1.1</version>
</dependency>
```

然后就可以直接new出一个ModelMapper对象，并且调用其map方法将指定对象的值映射到另一个对象上了。



使用方法今天不做过多介绍，大家可以自行Google，找到ModelMapper的相关文档进行学习。今天要分享的时前几天无意间踩到的一个坑。我有两个类，PostDO和PostVO（这里只截取了部分字段，因此两个类的含义也不做解释了）：

``` java
public class PostDO {
    private Long id;
    private String commentId;
    private Long postId;
    private int likeNum;
}
```

``` java
public class PostVO {
    private Long id;
    private boolean like;
    private int likeNum;
}
```

在一个方法中，我试图将PostDO的一个对象映射到PostVO，因此我进行如下操作：

``` java
public class Application {
    public static void main(String[] args) {
        ModelMapper modelMapper = new ModelMapper();
        PostDO postDO = PostDO.builder().id(3L).likeNum(0).build();
        PostVO postVO = modelMapper.map(postDO, PostVO.class);
        System.out.println(postVO);
    }
}
```

执行结果是这样的：

``` java
PostVO(id=3, like=false, likeNum=0)
```

无异常，项目中likeNum字段的值是随着项目的进行递增的。当likeNum增加到2时，异常出现了：

``` java
Exception in thread "main" org.modelmapper.MappingException: ModelMapper mapping errors:

1) Converter org.modelmapper.internal.converter.BooleanConverter@497470ed failed to convert int to boolean.

1 error
	at org.modelmapper.internal.Errors.throwMappingExceptionIfErrorsExist(Errors.java:380)
	at org.modelmapper.internal.MappingEngineImpl.map(MappingEngineImpl.java:79)
	at org.modelmapper.ModelMapper.mapInternal(ModelMapper.java:554)
	at org.modelmapper.ModelMapper.map(ModelMapper.java:387)
	at Application.main(Application.java:7)
Caused by: org.modelmapper.MappingException: ModelMapper mapping errors:

1) Error mapping 2 to boolean

1 error
	at org.modelmapper.internal.Errors.toMappingException(Errors.java:258)
	at org.modelmapper.internal.converter.BooleanConverter.convert(BooleanConverter.java:49)
	at org.modelmapper.internal.converter.BooleanConverter.convert(BooleanConverter.java:27)
	at org.modelmapper.internal.MappingEngineImpl.convert(MappingEngineImpl.java:298)
	at org.modelmapper.internal.MappingEngineImpl.map(MappingEngineImpl.java:108)
	at org.modelmapper.internal.MappingEngineImpl.setDestinationValue(MappingEngineImpl.java:238)
	at org.modelmapper.internal.MappingEngineImpl.propertyMap(MappingEngineImpl.java:184)
	at org.modelmapper.internal.MappingEngineImpl.typeMap(MappingEngineImpl.java:148)
	at org.modelmapper.internal.MappingEngineImpl.map(MappingEngineImpl.java:113)
	at org.modelmapper.internal.MappingEngineImpl.map(MappingEngineImpl.java:70)
	... 3 more
```

提示int类型不能转换成boolean型，很明显。ModelMapper是将like字段映射到likeNum了。那么ModelMapper究竟是怎样进行映射的呢，我们一起来看一下ModelMapper的源码。

ModelMapper利用反射机制，获取到目标类的字段，并生成期望匹配的键值对，类似于这样。

![键值对](https://res.cloudinary.com/dxydgihag/image/upload/v1536677296/Blog/ModelMapper/ModelMapper1.png)

接着对这些键值对进行遍历，逐个寻找源类中可以匹配的字段。首先会根据目标字段判断是否存在对应的映射，

``` java
Mapping existingMapping = this.typeMap.mappingFor(destPath);
if (existingMapping == null) {
    this.matchSource(this.sourceTypeInfo, mutator);
    this.propertyNameInfo.clearSource();
    this.sourceTypes.clear();
}
```

如果不存在，就调用matchSource方法，在源类中根据匹配规则寻找可以匹配的字段。匹配过程中，首先会判断目标字段的类型是否在类型列表中存在，如果存在，则可以根据名称，加入匹配的mappings中。如果不存在，则需要判断converterStore中是否存在能够应用于该字段的转换器。

``` java
if (this.destinationTypes.contains(destinationMutator.getType())) {
    this.mappings.add(new PropertyMappingImpl(this.propertyNameInfo.getSourceProperties(), this.propertyNameInfo.getDestinationProperties(), true));
} else {
    TypeMap<?, ?> propertyTypeMap = this.typeMapStore.get(accessor.getType(), destinationMutator.getType(), (String)null);
    PropertyMappingImpl mapping = null;
    if (propertyTypeMap != null) {
        Converter<?, ?> propertyConverter = propertyTypeMap.getConverter();
        if (propertyConverter == null) {
            this.mergeMappings(propertyTypeMap);
        } else {
            this.mappings.add(new PropertyMappingImpl(this.propertyNameInfo.getSourceProperties(), this.propertyNameInfo.getDestinationProperties(), propertyTypeMap.getProvider(), propertyConverter));
        }

        doneMatching = this.matchingStrategy.isExact();
    } else {
        Iterator var9 = this.converterStore.getConverters().iterator();

        while(var9.hasNext()) {
            ConditionalConverter<?, ?> converter = (ConditionalConverter)var9.next();
            MatchResult matchResult = converter.match(accessor.getType(), destinationMutator.getType());
            if (!MatchResult.NONE.equals(matchResult)) {
                mapping = new PropertyMappingImpl(this.propertyNameInfo.getSourceProperties(), this.propertyNameInfo.getDestinationProperties(), false);
                if (MatchResult.FULL.equals(matchResult)) {
                    this.mappings.add(mapping);
                    doneMatching = this.matchingStrategy.isExact();
                    break;
                }

                if (!this.configuration.isFullTypeMatchingRequired()) {
                    this.partiallyMatchedMappings.add(mapping);
                    break;
                }
            }
        }
    }

    if (mapping == null) {
        this.intermediateMappings.put(accessor, new PropertyMappingImpl(this.propertyNameInfo.getSourceProperties(), this.propertyNameInfo.getDestinationProperties(), false));
    }
}
```

默认的转换器有11中：

![转换器](https://res.cloudinary.com/dxydgihag/image/upload/v1536678691/Blog/ModelMapper/ModelMapper2.png)

找到对应的converter后，converter的map方法返回一个MatchResult，MatchResult有三种结果：FULL、PARTIAL和NONE（即全部匹配，部分匹配和不匹配）。注意，这里有一个部分匹配，也就是我所踩到的坑，在对like进行匹配是，likeNum就被定义为部分匹配。因此，当likeNum大于2时，就不能被转换成boolean类型。



这里解决方法有两种，一种是在设置中，规定必须字段名完全匹配；另一种就是将匹配策略定义为严格。

设置方法如下：

``` java
modelMapper.getConfiguration().setFullTypeMatchingRequired(true);
modelMapper.getConfiguration().setMatchingStrategy(MatchingStrategies.STRICT);
```

到这里，ModelMapper会选出较为合适的源字段，但是如果匹配要求不高的话，ModelMapper可能会筛选出多个符合条件的字段，因此，还需要进一步过滤。

``` java
PropertyMappingImpl mapping;
if (this.mappings.size() == 1) {
    mapping = (PropertyMappingImpl)this.mappings.get(0);
} else {
    mapping = this.disambiguateMappings();
    if (mapping == null && !this.configuration.isAmbiguityIgnored()) {
        this.errors.ambiguousDestination(this.mappings);
    }
}
```

这里我们看到，如果匹配到的结果只有1个，那么就返回这个结果；如果有多个，则会调用disambiguateMappings方法，去掉有歧义的结果。我们来看一下这个方法。

``` java
private PropertyMappingImpl disambiguateMappings() {
    List<ImplicitMappingBuilder.WeightPropertyMappingImpl> weightMappings = new ArrayList(this.mappings.size());
    Iterator var2 = this.mappings.iterator();

    while(var2.hasNext()) {
        PropertyMappingImpl mapping = (PropertyMappingImpl)var2.next();
        ImplicitMappingBuilder.SourceTokensMatcher matcher = this.createSourceTokensMatcher(mapping);
        ImplicitMappingBuilder.DestTokenIterator destTokenIterator = new ImplicitMappingBuilder.DestTokenIterator(mapping);

        while(destTokenIterator.hasNext()) {
            matcher.match(destTokenIterator.next());
        }

        double matchRatio = (double)matcher.matches() / ((double)matcher.total() + (double)destTokenIterator.total());
        weightMappings.add(new ImplicitMappingBuilder.WeightPropertyMappingImpl(mapping, matchRatio));
    }

    Collections.sort(weightMappings);
    if (((ImplicitMappingBuilder.WeightPropertyMappingImpl)weightMappings.get(0)).ratio == ((ImplicitMappingBuilder.WeightPropertyMappingImpl)weightMappings.get(1)).ratio) {
        return null;
    } else {
        return ((ImplicitMappingBuilder.WeightPropertyMappingImpl)weightMappings.get(0)).mapping;
    }
}
```

ModelMapper定义了一个权重，来判断源字段是否有歧义，这里根据驼峰式的规则（也可以设置为下划线），将源和目标字段名称进行拆分，根据 匹配数量/源token数+目标token数，得到一个匹配的比率，比率越大，说明匹配度越高。最终取得匹配权重最大的那个字段。其他字段被认为是有歧义的。



截至目前，默认的ModelMapper的map方法的工作原理已经介绍完了，中间可能有些遗漏的细节，或者哪里有说的不明白的地方，欢迎大家和我一起讨论。大家在用到ModelMapper时一定要注意字段名，如果有相近的字段名，必须认真核对匹配是否正确，必要时就采用严格匹配策略。