---
title: Redis命令详解：Geo
date: 2019-09-06 00:16:49
tags: Redis命令
---

Redis Geo相关命令。<!-- more -->

#### GEOADD

最早可用版本：3.2.0

时间复杂度：O(log(N))，N是Sorted set元素数量

用法：GEOADDkey longitude latitude member [longitude latitude member ...]

将指定的地理空间位置（纬度、经度、名称）添加到指定key中。这些数据将存储到sorted set，这样为了方便使用GEORADIUS或GEORADIUSBYMEMBER命令。

该命令采用标准格式参数x,y，所以经度必须在纬度之前。输入的坐标有如下限制：

- 有效的经度从-180度到180度
- 有效的纬度从-85.05112878度到85.05112878度

当坐标位置超出上述指定范围时，该命令返回一个错误。

#### GEODIST

最早可用版本：3.2.0

时间复杂度：O(log(N))

用法：GEODIST key member1 member2 [unit]

返回两个给定位置之间的距离。

如果两个位置之间的其中一个不存在，那么命令返回空值。

指定单位的参数unit必须是以下其中一个：

- m表示单位为米
- km表示单位为千米
- mi表示单位为英里
- ft表示单位为英尺

如果用户没有显示指定单位参数，默认使用米作为单位。

GEODIST命令在计算距离时会假设地球为完美球形，极限情况下，这一假设最大会造成0.5%的误差。

#### GEOHASH

最早可用版本：3.2.0

时间复杂度：O(log(N))

返回一个或多个元素位置的Geohash表示。

用法：GEOHASH key member [member ...]

返回一个或多个位置元素的Geohash表示。

通常，Redis使用Geohash技术的变体表示元素的位置，位置使用52位整数进行编码。由于编码和解码过程的初始最大和最小坐标不同，所以编码也不是标准的编码方式。

该命令返回11个字符的Geohash字符串，和内部的52位表示方法相比没有精度的损失。返回的Geohash有以下属性：

1. 它可以移除右边的字符以缩短长度，这只会导致精度的损失，但仍指向同一区域
2. 它可以在heohash.org网站使用，地址是http://geohash.org/<geohash-string>
3. 前缀相似的字符串指向的位置离得很近，但这不代表前缀不同的字符串就离得很远

#### GEOPOS

最早可用版本：3.2.0

时间复杂度：O(log(N))

用法：GEOPOS key member [member ...]

返回指定key中的指定位置信息。

#### GEORADIUS

最早可用版本：3.2.0

时间复杂度：O(N+log(M))，N是半径区域内元素数量，M是指定key中元素数量

用法：GEORADIUS key longitude latitude radiusm|km|ft|mi \[WITHCOORD\] \[WITHDIST\] \[WITHHASH\]\[COUNT count\] \[ASC|DESC\] \[STORE key\]\[STOREDIST key\]

以给定经纬度为中心，返回键包含的位置元素与中心距离不超过最大距离的所有位置元素。

命令额外选项：

- WITHDIST：在返回位置元素的同时，将位置元素与中心的距离也一并返回，单位与用户给定距离的单位一直
- WITHCOORD：将位置元素的经度和纬度也一并返回
- WITHHASH：以52位有符号整数的形式，返回位置元素经过原始geohash编码的有序集合分值。这个选项主要用于底层应用或调试。

命令默认返回结果未排序，可以指定ASC或DESC按距离排序。

COUNT表示指定返回元素的数量，如果不指定则返回全部符合的元素。

当GEORADIUS和GEORADIUSBYMEMBER命令有了STORE和STOREDIST参数时，这两命令被标记成了写命令。在集群中，如果设置了READONLY，它们将被重定向到主节点，即使它们没有做写操作。但为了解决这个问题，在Redis4.0引入了这两个命令的变种，分别是GEORADIUS_RO和GEORADIUSBYMEMBER_RO。

#### GEORADIUSBYMEMBER

最早可用版本：3.2.0

时间复杂度：O(N+log(M))，N是半径区域内元素数量，M是指定key中元素数量

用法：GEORADIUSBYMEMBER key member radiusm|km|ft|mi \[WITHCOORD\] \[WITHDIST\] \[WITHHASH\]\[COUNT count\] \[ASC|DESC\] \[STORE key\]\[STOREDIST key\]

这个命令和GEORADIUS命令一样，都可以找出位置范围内的元素，但指定中心点的方式不同，该命令直接指定key中的元素作为中心，而不像GEORADIUS一样指定经纬度。

