---
title: Redis Lua脚本小学教程
date: 2019-05-13 22:54:01
tags: Redis
comments: true
---

Redis提供了丰富的指令集，但是仍然不能满足所有场景，在一些特定场景下，需要自定义一些指定来完成某些功能。因此，Redis提供了Lua脚本支持，用户可以自己编写脚本来实现想要的功能。<!-- more -->

#### 什么是Lua？

Lua是一种功能强大的，高效，轻量级，可嵌入的脚本语言。它是动态类型语言，通过使用基于寄存器的虚拟机解释字节码运行，并具有增量垃圾收集的自动内存管理，是配置，脚本和快速原型设计的最佳选择。

#### Redis怎么执行Lua脚本

##### EVAL命令

Redis中可以使用EVAL命令执行相应的Lua脚本

```bash
> EVAL 'local val="Hello Jackey" return val' 0
"Hello Jackey"
```

你可以像这样在交互模式下执行Lua脚本，这样更方便处理错误。只是这样还不够，有时候，我们需要给Lua脚本传入一些参数。细心的同学一定注意到了，脚本的后面还有一个数字0，它的意思的不传入参数。

那怎么传参数呢？

```bash
> EVAL 'local val=KEYS[1] return val.." "..ARGV[1]' 1 Hello Redis
"Hello Redis"
```

其实也很简单，传入的参数都是kv形式的，这个数字代表传入参数的key的数量，再后面就是n个key和n个value。在脚本中，可以理解为从KEYS数组和ARGV数组中获取对应的值，下标是从1开始的。

*上面例子中的两个点是Lua脚本中字符串连接的操作符*

现在我们已经知道怎么在Redis中执行Lua脚本了，可是这样的脚本和Redis没有关系啊，怎么才能操作Redis中的数据呢？请继续看我表演

```bash
> get my_name
"Jackeyzhe"
> EVAL 'local val=ARGV[1].." "..redis.call("get",KEYS[1]) return val' 1 my_name Hello
"Hello Jackeyzhe"
```

使用redis.call或redis.pcall（以后会提到）就可以操作redis了。

需要注意的是，如果返回下面的错误，说明要获取的key不存在

```bash
> EVAL 'local val=ARGV[1].." "..redis.call("get",KEYS[1]) return val' 1 me Hello
(error) ERR Error running script (call to f_eb11f8ddeeee07cc88d1f3bd103069284b83c5d8): @user_script:1: user_script:1: attempt to concatenate a boolean value
```

我们可以使用上面这种方法执行一些简单的Lua脚本，如果要执行更加复杂的Lua脚本，用EVAL命令就会显得臃肿且凌乱。所以Redis又提供了一种方法。

##### redis-cli --eval

我们可以先写一个Lua文件，然后使用redis-cli命令来执行。

``` lu
local name=redis.call("get", KEYS[1])
local greet=ARGV[1]
local result=greet.." "..name
return result
```

```bash
> redis-cli --eval hello.lua my_name , Hello
"Hello Jackey"
```

这样，我们就可以先写一个.lua文件，然后再使用redis-cli命令来执行了，看起来也不会很凌乱，使用这种方式传入参数时，不需要指定key的数量，而是用逗号分隔key和argv。

##### EVALSHA

你以为到这就结束了吗？那就too naive了。如果我们在Redis交互模式中，想要执行脚本文件怎么办？每次都退出来，执行完再连接一次？这未免太麻烦了。Redis提供了EVALSHA命令，使我们可以在交互模式执行脚本文件。

首先，需要上传脚本文件

``` bash
$ redis-cli SCRIPT LOAD "$(cat hello.lua)"
"463ff2ca9e78e36cd66ee9d37ee0dcd59100bf46"
```

会得到一串十六进制的数字，这是这个脚本的唯一标识。拿到这个数字后，表示我们已经将脚本上传到服务器了，接下来就可以使用这个标识来执行脚本了。

``` bash
> EVALSHA 463ff2ca9e78e36cd66ee9d37ee0dcd59100bf46 1 my_name Hello
"Hello Jackeyzhe"
```

#### 终止脚本

Redis中Lua脚本到默认执行时长是5秒，一般情况下脚本的执行时间都是毫秒级的，如果执行超时，脚本也不会停止，而是记录错误日志。

终止脚本执行的方法有两种

1. 使用KILL SCRIPT命令
2. 使用SHUTDOWN NOSAVE命令关闭服务器

不过不建议手动终止脚本

#### 总结

本文简要介绍了什么是Lua，以及Redis执行和终止Lua脚本的方法。如果都掌握了，那么恭喜你已经从Lua小学毕业了。在Lua中学你会学到Redis关于Lua命令的更详细介绍。