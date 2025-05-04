---
title: Redis命令详解：Strings
date: 2018-10-07 12:04:38
tags: Redis命令
---

String类型是Redis中比较常用的类型，因此，和String相关的命令也比较多<!-- more -->

#### APPEND

最早可用版本2.0.0

当指定的key存在，并且value是字符串时，APPEND命令会在字符串末尾追加指定的字符串，如果指定的key不存在，则会创建一个空的字符串，并且追加上指定的value，效果类似于SET命令。

该命令的返回值是执行后字符串的长度。

``` bash
127.0.0.1:6379> EXISTS mykey
(integer) 0
127.0.0.1:6379> APPEND mykey Jackeyzhe
(integer) 9
127.0.0.1:6379> APPEND mykey 2018
(integer) 13
127.0.0.1:6379> GET mykey
"Jackeyzhe2018"
```

APPEND常被用作为定长的数据提供紧凑的存储。可以通过GETRANGE命令来获取指定长度范围的字符串，这里推荐使用Unix的时间戳作为key，既不会因为单个key过大而影响效率，又节省了大量命名空间。



#### BITCOUNT

最早可用版本2.6.0

该命令的时间复杂度是O(N)，用来统计字符串中被设置为1的比特数。默认检查整个字符串，当然也可以指定起始和结束位置。起始和结束位置可以是负数，例如-1表示最后一个字节，-2表示倒数第二字节，以此类推。

``` bash
127.0.0.1:6379> SETBIT bitkey 0 1   #0001
(integer) 0
127.0.0.1:6379> BITCOUNT bitkey
(integer) 1
127.0.0.1:6379> SETBIT bitkey 2 1   #0101
(integer) 0
127.0.0.1:6379> BITCOUNT bitkey
(integer) 2
```

这个命令可以用来统计实时的数据。例如，统计用户上线历史，我们可以使用用户名作为key，如果第n天上线，将对应的第n位置为1。这样，即使统计10年的数据，每个用户所使用的内存空间仅仅是456字节。对于这样的数据量来讲，BITCOUNT处理的速度和其他时间复杂度为O(1)的命令是一个数量级的。



#### BITFIELD

最早可用版本3.2.0

用法：BITFIELD key \[GET type offset\] \[SET type offset value\] \][INCRBY type offset increment\] \[OVERFLOW WRAP|SAT|FAIL\]

这个命令把Redis的字符串看作是一个bit数组。可以把指定偏移位置的bit当做指定的类型处理。例如，以下命令是对偏移量100的8位有符号整数增1，获取偏移量为0的4位无符号整数。

``` bash
> BITFIELD mykey INCRBY i8 100 1 GET u4 0
1) (integer) 1
2) (integer) 0
```

该命令支持的子命令有：

- GET<type><offset>
- SET<type><offset><value>
- INCRBY<type><offset><increment>

另外还有一个OVERFLOW命令用来进行INCRBY后的益处控制。下面是OVERFLOW的三种控制方法，默认为WRAP算法。

- WRAP：回环算法，适用于有符号和无符号两种类型。对于无符号整型，回环计数将对整型最大值进行取模操作；对有符号整数，上溢从最小负数开始，下溢从最大正数开始。例如，i8最大为127，加1后变成-128。
- SAT：饱和算法，上溢后保持最大整数，下溢后保持最小整数。
- FAIL：失败算法，这种模式下发生上溢或者下溢，不会做任何操作，返回值为NULL。

该命令的偏移量有两种指定方式，如果是不带前缀的数字，则以字符串位计算，如果数字前有#前缀，则计算偏移量时应该指定数字乘以整型宽度。



#### BITOP

最早可用版本：2.6.0

时间复杂度：O(N)

用法：BITOP operation destkey key [key ...]

对一个或者字符串进行位操作，支持与(AND)、或(OR)、非(NOT)、异或(XOR)操作。除了非操作，其他的都支持多个key作为输入。对于长度不同的字符串，较短的字符串缺少的部分会以0补齐，空key也会被看作全部为0的字符串序列。

该命令返回保存到destkey的字符串长度，也就是输入字符串的最大长度。

``` bash
127.0.0.1:6379> set key1 "abcde"
OK
127.0.0.1:6379> set key2 "abcd"
OK
127.0.0.1:6379> BITOP and dest key1 key2
(integer) 5
127.0.0.1:6379> get dest
"abcd\x00"
```



#### BITPOS

最早可用版本：2.8.7

时间复杂度：O(N)

用法：BITPOS key bit \[start\]\[end\]

该命令用于返回第一个被设置为0或1的位置。可以使用start和end参数指定查询范围，需要注意的是，这个范围指的是字节范围而不是位范围，也就是说start=0，end=2表示在前三个字节中查找。start和end都可以为负值，-1表示最后一位，-2表示倒数第二位，以此类推。

我们通过一些例子来看一下某些特殊情况下的返回值。

```bash
127.0.0.1:6379> set mykey ""
OK
127.0.0.1:6379> get mykey
""
127.0.0.1:6379> bitpos mykey 0
(integer) -1
127.0.0.1:6379> bitpos mykey 1
(integer) -1
127.0.0.1:6379> set key1 "\xff"
OK
127.0.0.1:6379> bitpos key1 1
(integer) 0
127.0.0.1:6379> bitpos key1 0
(integer) 8
```

如果是空字符串，那么查找0和1都会返回-1。如果是类似"\xff"这样的字符串，它的0-7位都是1，如果查询0时，会返回再往右数一位也就是第8位。



#### DECR

最早可用版本：1.0.0

时间复杂度：O(1)

对指定的key进行减1操作，操作数最大为64位有符号整数。如果key不存在，则会先将其设置为0，如果类型不符合，则会抛出错误。

``` bash
127.0.0.1:6379> GET unexist
(nil)
127.0.0.1:6379> DECR unexist
(integer) -1
127.0.0.1:6379> SET mykey "fdsfe"
OK
127.0.0.1:6379> DECR mykey
(error) ERR value is not an integer or out of range
```



#### DECRBY

最早可用版本：1.0.0

时间复杂度：O(1)

这个命令与DECR的参数要求和使用方法相同，唯一不同的是它用来减去指定的数值。



#### GET

最早可用版本：1.0.0

时间复杂度：O(1)

这个不做过多介绍，是最常用的命令之一。返回指定key的值，如果不是字符串，就返回错误。



#### GETBIT

最早可用版本：2.2.0

时间复杂度：O(1)

返回指定偏移量位的bit值，当key不存在时，返回0。



#### GETRANGE

最早可用版本：2.4.0

时间复杂度：O(N)

用法：GETRANGE key start end

这个命令在Redis2.0之前叫做SUBSTR，返回指定的key的指定范围（包含start和end）的子串。start和end同样也可以是负数，这点可以参考BITPOS命令。



#### GETSET

最早可用版本：1.0.0

时间复杂度：O(1)

自动把新的value保存到指定key中，并且返回旧的value。如果key存在，但是保存的数据不是字符串则会报错。



#### INCR

最早可用版本：1.0.0

时间复杂度：O(1)

该命令用于对指定key进行加1操作，与DECR命令正好相反。执行此操作时，字符串被解析为10进制的64位有符号整数。由于Redis内部有整数形式（integer representation）来保存整数，因此不会有整数存储为字符串的额外开销。



#### INCRBY

最早可用版本：1.0.0

时间复杂度：O(1)

它与INCR命令的关系就像DECR命令和DECRBY命令的关系一样，只是指定了要加的数值。



#### INCRBYFLOAT

最早可用版本：2.6.0

时间复杂度：O(1)

该命令会把字符串解析为浮点数，然后加上指定的浮点数。如果value不是字符串类型或者不能解析为浮点数，则会报错。返回值的精度为小数点后17位。其内部以科学计数法的形式存储。



#### MGET

最早可用版本：1.0.0

时间复杂度：O(N)，N为取回key的个数

该命令返回多个key的值，对于不是string类型或者不存在的key，都返回nil。



#### MSET

最早可用版本：1.0.1

时间复杂度：O(N)，N为需要设置的key的个数

设置所有的key，如果已经存在，则覆盖旧值。MSET命令是原子操作，并且不会失败。



#### MSETNX

最早可用版本：1.0.1

时间复杂度：O(N)，N为需要设置的key的个数

设置所有的key，如果有一个key已经存在，则所有的key都会设置不成功。返回1表示所有的key都已经设置成功，返回0表示所有的key都没有设置成功。



#### PSETEX

最早可用版本：2.6.0

时间复杂度：O(1)

用法：PSETEX key milliseconds value

该命令类似于SETEX（在后面介绍），唯一不同的时，该命令设置过期时间以毫秒为单位。



#### SET

最早可用版本：1.0.0

时间复杂度：O(1)

用法：SET key value \[EX seconds\]\[PX milliseconds\]\[NX|XX\]

也是最常用的命令之一。在2.6.12版本，SET命令加上了一些参数：

- EX seconds – 设置键key的过期时间，单位时秒
- PX milliseconds – 设置键key的过期时间，单位时毫秒
- NX – 只有键key不存在的时候才会设置key的值
- XX – 只有键key存在的时候才会设置key的值

加上这参数之后，SET命令已经取代了SETNX、SETEX、PSETEX这三个命令，因此，Redis不再推荐使用这些命令，并且有可能在未来版本中抛弃这些命令。



#### SETBIT

最早可用版本：2.2.0

时间复杂度：O(1)

用法：SETBIT key offset value

设置指定位置的bit值。当key不存在时，会先生成一个字符串，这个字符串必须保证offset处有值。offset必须大于0，小于2<sup>32</sup>（因为bitmap的大小限制为512M）。



#### SETEX

最早可用版本：2.0.0

时间复杂度：O(1)

为指定key设置value，并且给定超时时间（单位是秒），SETEX是原子操作。



#### SETNX

最早可用版本：1.0.0

时间复杂度：O(1)

SETNX是"SET if Not Exist"的缩写，也就是说，当key不存在时，才会SET成功，成功返回1，失败返回0。



#### SETRANGE

最早可用版本：2.2.0

时间复杂度：O(1)

这个命令用来覆盖key的一部分内容，如果offset超出value的长度，则会为string补0。offset最大值是2<sup>29</sup> -1 (536870911)。

``` bash
127.0.0.1:6379> SET follow jackey
OK
127.0.0.1:6379> SETRANGE follow 8 lol
(integer) 11
127.0.0.1:6379> GET follow
"jackey\x00\x00lol"
```



#### STRLEN

最早可用版本：2.2.0

时间复杂度：O(1)

返回指定key存储的value的长度，如果value不是字符串，则会报错。如果key不存在，则返回0。