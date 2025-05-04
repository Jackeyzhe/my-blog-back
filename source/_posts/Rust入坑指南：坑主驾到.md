---
title: Rust入坑指南：坑主驾到
date: 2019-09-21 21:41:35
tags: Rust
---

欢迎大家和我一起入坑Rust，以后我就是坑主，我主要负责在前面挖坑，各位可以在上面看，有手痒的也可以和我一起挖。这个坑到底有多深？我也不知道，我是抱着有多深就挖多深的心态来的，下面我先跳了，各位请随意。<!-- more -->

### Rust简介

众所周知，在编程语言中，更易读的高级语言和控制底层资源的低级语言是一对矛盾体。Rust想要挑战这一现状，它尝试为开发者提供更好的体验的同时给予开发者控制底层细节的权限（比如内存使用）。

低级语言在开发过程中很容易出现各种细微的错误，它们难以发现但是可能影响巨大。其他大部分低级语言只能靠覆盖面更广的测试用例和经验丰富的开发者来解决这些问题。而Rust则依靠严格的编译器来杜绝这些问题。

*Ps：以后会见识到Rust编译器的「厉害」*

Rust的一些工具：

- Cargo，依赖包的管理和构建工具，可以帮你减轻添加、编译和管理依赖包的痛苦
- Rustfmt，用于保证开发者代码风格的一致性
- Rust语言服务器支持集成IDE（我用的是IDEA）

### 安装Rust

如果你的操作系统是Linux或macOS，在终端执行命令

```bash
$ curl https://sh.rustup.rs -sSf | sh
```

安装过程中的选项使用默认就好（一路回车），直到出现以下信息时，表示安装成功。

```bash
Rust is installed now. Great!
```

安装脚本会自动把Rust添加到环境变量PATH中，可以重启终端或者手动执行命令使添加生效。

```bash
$ source $HOME/.cargo/env
```

当然也可以添加到你的.bash_profile文件中：

```bash
$ export PATH="$HOME/.cargo/bin:$PATH"
```

最后，执行以下命令来检查Rust是否安装成功

```bash
$ rustc --version
```

另外，当你尝试编译Rust代码，但报了linker不可执行的错误时，你需要手动安装一个linker，C编译器通常会包含正确的linker。Rust的一些公共包也会依赖C语言代码和编译器。所以最好现在安装一个。

#### IDEA集成Rust

IDEA中集成Rust也很简单，只需要在Preference->Plugins中搜索Rust，安装Rust插件后重启IDEA就可以了。

### Hello World

又到了经典的Hello World时间，这次我不想直接一个简单的print就结束了，我们一开始提到了Cargo是Rust依赖包的管理工具，所以我想体验一下Cargo的用法。

首先新建一个项目，可以直接用在IDEA中new project，也可以使用Cargo命令

```bash
cargo new hello-world
cd hello-world
```

新建好项目以后，它的结构长这样子

![rust-new-project](https://res.cloudinary.com/dxydgihag/image/upload/v1569055727/Blog/rust/01/rust01.png)

其中

- main.rs是我们代码的入口文件
- Cargo.toml是记录Rust元数据的文件，包括依赖。
- Cargo.lock是记录增加依赖log的文件，不能手动修改。

接着我们在Cargo.toml文件中添加我们需要的依赖

```bash
[dependencies]
ferris-says = "0.1"
```

这时IDEA会自动安装依赖包，如果没有安装，也可以手动执行命令来安装

```bash
cargo build
```

依赖安装好以后，就可以开始写代码了：

```rust
use ferris_says::say;
use std::io::{stdout, BufWriter};

fn main() {
    let stdout = stdout();
    let out = b"Hello World!";
    let width = 12;

    let mut writer = BufWriter::new(stdout.lock());
    say(out, width, &mut writer).unwrap();
}
```

执行结果

```bash
----------------
| Hello World! |
----------------
              \
               \
                  _~^~^~_
              \) /  o o  \ (/
                '_   -   _'
                / '-----' \
```

没错，这是一个小螃蟹，至于它是谁，来看看官方解释

> Ferris is the unofficial mascot of the Rust Community. Many Rust programmers call themselves “Rustaceans,” a play on the word “[crustacean](https://en.wikipedia.org/wiki/Crustacean).” We refer to Ferris with the pronouns “they,” “them,” etc., rather than with gendered pronouns.
>
> Ferris is a name playing off of the adjective, “ferrous,” meaning of or pertaining to iron. Since Rust often forms on iron, it seemed like a fun origin for our mascot’s name!
>
> You can find more images of Ferris on http://rustacean.net/.

关于toml文件可能有些读者不太熟悉（其实我自己也不太熟），这里简单介绍一下吧，它的全称是「Tom's Obvious, Minimal Language」，是一种配置文件格式。它的语义是比较明显的，因此易于阅读。同时格式可以明确的映射到hash表，所以也可以被多种语言轻松解析。

GitHub地址是：https://github.com/toml-lang/toml

有兴趣的同学可以做更深入的了解。

### 后记

至此，我确信自己已经跳进来了，有想跟进的朋友记得关注我哦。