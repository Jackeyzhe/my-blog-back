---
title: 【译】教你用16个小时从0构建一个Rust应用
date: 2020-02-23 23:26:43
tags: 技术杂谈
---

我们在2019年的最后两天，参加了Prodigy Education举办的黑客马拉松，许多团队聚在一起努力将他们的想法变成现实。<!-- more -->

我们之中有的人只是单纯为了好玩，有的是想学一些新的知识，还有些人可能是想证明一些概念或想法。

我在过去几周总是被动的获取Rust相关信息或使用Rust的代码，因此我认为hackathon是一次学习Rust的绝佳时机。

hackathon的时间紧迫性使我更加快速的去学习，同时也会去解决现实世界的一些问题。

### 为什么是Rust

![Getting a chance to peek under the hood again](https://res.cloudinary.com/dxydgihag/image/upload/v1582370825/Blog/Other/learn_rust/lr-2.jpg)

在我职业生涯的前10年中，有8年都在使用C和C++。

从好的方面来讲，我喜欢像C++这样可以提供静态类型的语言，因为它能在编译期就能够早早的发现错误。

我个人对于C++的一些看法是：

- 工程师很容易搬起石头砸自己的脚
- 作为一门编程语言，它已经非常臃肿且复杂
- 缺乏良好的、标准的广泛适用的包管理系统

自从我改做Web应用以来，一直是做Python和JavaScript开发，使用像Django、Flask和Express这样的框架。

到目前为止，我在Python和JavaScript中的开发经验是，它们可以提供良好的程序迭代和交付速度，但有时会占用大量的CPU和内存，即使服务是相对空闲的。

我经常发现自己写好的C++程序，会缺失一些安全性、速度和精简性。

我想要寻找一种像Rust这样精简的、裸机编程语言来开发web应用。

没有运行时，没有垃圾回收。直接加载二进制代码，交给内核执行。

### 目标

我的目标是完成一个后端由Rust编写，前端是JavaScript+React完成的类似于S3作为图床的应用程序，用户可以做以下事情：

- 浏览图床中所有的图片（分页可选）
- 上传图片
- 上传图片时可以给图片增加标签
- 通过名称进行查询或过滤

所有有趣的hackathon项目都有一个名字，所以我决定将这个项目命名为：

RustIC -> Rust + Image Contents

![Let’s hack something great](https://res.cloudinary.com/dxydgihag/image/upload/v1582370820/Blog/Other/learn_rust/lr3.jpg)

我认为如果我做到了以下这些事情，那么这次hackathon之行对我个人来说就是成功的：

- 对Rust有一个基本的理解，包括它的类型系统和内存模型
- 探索S3的对于文件和任意标签的预签名链接功能
- 写出一个可以验证的功能正常的应用

由于我的主要目标是开发功能，同时兼顾学习。很多代码是我一边学一边写的，所以代码组织和效率可能并不是最理想的，因为这些属于次要目标。

### Rust的原则

在我开始之前，我带着好奇心去了解了要学习的语言的设计师在创建这门语言时内心的原则是什么。我找到了一个[简化版本](https://doc.rust-lang.org/1.4.0/complement-design-faq.html)和一个[详细版本](https://github.com/dtolnay/rust-faq)。

与我在许多博客上读到的内容相反，Rust是有可能发生内存泄露（循环引用）和之行不安全的操作（unsafe代码块中）的，详细描述在上面的FAQ中。

> *“We [the language creators] do not intend [for Rust] to be 100% static, 100% safe, 100% reflective.”*

![Dazzling, intricate, sophisticated](https://res.cloudinary.com/dxydgihag/image/upload/v1582370827/Blog/Other/learn_rust/lr-4.jpg)

### 从后端开始

Google搜索“Rust web framework“，排在最前面的是[Rocket](https://rocket.rs/)。我进入这个网站，发现文档的示例都一目了然。

有一点需要注意的是Rocket需要Rust的nightly版本，不过在hackathon上这都是小问题。

GitHub的[代码库](https://github.com/SergioBenitez/Rocket/tree/v0.4)中有着非常丰富的例子。完美！

我使用[Cargo](https://doc.rust-lang.org/cargo/)创建了一个新的项目，在TOML文件中加入了Rocket依赖，然后跟着Rocket的[入门指南](https://rocket.rs/v0.4/guide/getting-started/)，写了第一段代码：

``` rust
#[get("/")]
fn index() -> &'static str {
    "Hello, world!"
}

fn main() {
    rocket::ignite().mount("/", routes![index]).launch();
}
```

对于熟悉Django、Flask、Express等框架等同学来说，这段代码读起来非常容易。作为一名Rocket用户，你可以使用宏作为装饰器来将路由映射到对应的处理函数上。

在编译时，宏将被扩展。这对开发者是完全透明的。如果你想看扩展后的代码，可以使用[cargo-expand](https://github.com/dtolnay/cargo-expand)。

以下是我在构建Rust应用程序时的一些有趣的或者有挑战性的亮点：

#### 指定路由响应

我想要以JSON的数据格式返回S3中所有的文件列表。

你可以看到路由关联的处理函数的代码决定了响应类型。

设置响应结构非常容易，如果你想要返回JSON格式的数据，并且每个字段都有自己的结构和类型，那对应的就是Rust的```struct```。

所以你应该先定义一个结构体`struct(S)`来接受响应，并且需要进行标注：

``` rust
#[derive(Serialize)]
```

struct(s)被标记了`#[derive(Serialize)]`，因此可以通过`rocket_contrib::json::Json将它转换成JSON`。

``` rust
#[derive(Serialize)]
struct BucketContents {
    data: Vec<S3Object>,
}

#[derive(Serialize)]
struct S3Object {
    file_name: String,
    presigned_url: String,
    tags: String,
    e_tag: String, // AWS generated MD5 checksum hash for object
    is_filtered: bool,
}

#[get("/contents?<filter>")]
fn get_bucket_contents(
    filter: Option<&RawStr>
) -> Result<Json<BucketContents>, Custom<String>> {
    // Returns either Ok(Json(BucketContents)) or,
    // a Custom error with a reason
}
```

#### 处理分段上传

当我意识到我的前端很有可能使用POST方法上传格式为`multipart/form-data`的表单数据时，我就开始深入研究如何使用Rocket来构建程序了。

不幸的是，Rocket0.4版本不支持multipart，看起来在0.5版本会支持。

这意味着我需要使用[multipart](https://crates.io/crates/multipart) crate并集成到Rocket中。最终代码可以正常运行，但是如果Rocket支持multipart将会使代码更加简洁。

``` rust
#[post("/upload", data = "<data>")]
// signature requires the request to have a `Content-Type`. The preferred way to handle the incoming
// data would have been to use the FromForm trait as described here: https://rocket.rs/v0.4/guide/requests/#forms
// Unfortunately, file uploads are not supported through that mechanism since a file upload is performed as a
// multipart upload, and Rocket does not currently (As of v0.4) support this. 
// https://github.com/SergioBenitez/Rocket/issues/106
fn upload_file(cont_type: &ContentType, data: Data) -> Result<Custom<String>, Custom<String>> {
    // this and the next check can be implemented as a request guard but it seems like just
    // more boilerplate than necessary
    if !cont_type.is_form_data() {
        return Err(Custom(
            Status::BadRequest,
            "Content-Type not multipart/form-data".into()
        ));
    }

    let (_, boundary) = cont_type.params()
                                 .find(|&(k, _)| k == "boundary")
                                 .ok_or_else(
        || Custom(
            Status::BadRequest,
            "`Content-Type: multipart/form-data` boundary param not provided".into()
        )
    )?;

    // The hot mess that ensues is some weird combination of the two links that follow
    // and a LOT of hackery to move data between closures.
    // https://github.com/SergioBenitez/Rocket/issues/106
    // https://github.com/abonander/multipart/blob/master/examples/rocket.rs
    let mut d = Vec::new();
    data.stream_to(&mut d).expect("Unable to read");
    let mut mp = Multipart::with_body(Cursor::new(d), boundary);

    let mut file_name = String::new();
    let mut categories_string = String::new();
    let mut raw_file_data = Vec::new();

    mp.foreach_entry(|mut entry| {
        if *entry.headers.name == *"fileName" { 
            let file_name_vec = entry.data.fill_buf().unwrap().to_owned();
            file_name = from_utf8(&file_name_vec).unwrap().to_string()
        } else if *entry.headers.name == *"tags" {
            let tags_vec = entry.data.fill_buf().unwrap().to_owned();
            categories_string = from_utf8(&tags_vec).unwrap().to_string();
        } else if *entry.headers.name == *"file" {
            raw_file_data = entry.data.fill_buf().unwrap().to_owned()
        }
    }).expect("Unable to iterate");

    let s3_file_manager = s3_interface::S3FileManager::new(None, None, None, None);
    s3_file_manager.put_file_in_bucket(file_name.clone(), raw_file_data);

    let tag_name_val_pairs = vec![("tags".to_string(), categories_string)];
    s3_file_manager.put_tags_on_file(file_name, tag_name_val_pairs);

    return Ok(
        Custom(Status::Ok, "Image Uploaded".to_string())
    );
}
```

#### 配置CORS

路由写好了以后，我就开始用curl或Postman来进行测试了，现在已经是时候开始把前端集成进来了。我需要适当设置响应头以避免跨域问题。

Rocket依旧没有支持这个特性。

然后我在GitHub代码库中找到了一些解决方案：

``` rust
// CORS Solution below comes from: https://github.com/SergioBenitez/Rocket/issues/25
extern crate rocket;

use std::io::Cursor;
use rocket::fairing::{Fairing, Info, Kind};
use rocket::{Request, Response};
use rocket::http::{Header, ContentType, Method};

struct CORS();

impl Fairing for CORS {
    fn info(&self) -> Info {
        Info {
            name: "Add CORS headers to requests",
            kind: Kind::Response
        }
    }

    fn on_response(&self, request: &Request, response: &mut Response) {
        if request.method() == Method::Options || 
           response.content_type() == Some(ContentType::JSON) || 
           response.content_type() == Some(ContentType::Plain) {

            response.set_header(Header::new("Access-Control-Allow-Origin", "http://localhost:3000"));
            response.set_header(Header::new("Access-Control-Allow-Methods", "POST, GET, OPTIONS"));
            response.set_header(Header::new("Access-Control-Allow-Headers", "Content-Type"));
            response.set_header(Header::new("Access-Control-Allow-Credentials", "true"));
        }

        if request.method() == Method::Options {
            response.set_header(ContentType::Plain);
            response.set_sized_body(Cursor::new(""));
        }
    }
}

fn main() {
    
    rocket::ignite().attach(
        CORS()
    ).mount(
        "/", 
        routes![get_bucket_contents, upload_file]
    ).launch();
}
```

过了一会，我发现了[rocket_cors](https://crates.io/crates/rocket_cors)，它帮助我大幅缩减了代码量。

``` rust
fn main() -> Result<(), Error> {
    let allowed_origins = AllowedOrigins::some_exact(&["http://localhost:3000"]);

    let cors = rocket_cors::CorsOptions {
        allowed_origins,
        allowed_methods: vec![Method::Get, Method::Post].into_iter().map(From::from).collect(),
        allowed_headers: AllowedHeaders::some(&["Content-Type", "Authorization", "Accept"]),
        allow_credentials: true,
        ..Default::default()
    }
    .to_cors()?;


    rocket::ignite().attach(cors)
                    .mount("/", routes![get_bucket_contents, upload_file])
                    .launch();

    Ok(())
}
```

#### 运行起来

我们只需要一个简单的`cargo run`命令就可以让程序运行起来

![output](https://res.cloudinary.com/dxydgihag/image/upload/v1582370830/Blog/Other/learn_rust/lr-5.png)

我机器上的活动监视器告诉我这个程序正在运行中，并且只消耗了2.7MB内存。

而且这还只是没有经过优化的调试版本。项目使用`- release`标签打包的话，运行时只需要1.6MB内存。

![memory](https://res.cloudinary.com/dxydgihag/image/upload/v1582370814/Blog/Other/learn_rust/lr-6.png)

基于Rust的后端服务器，我们请求`/contents`这个路由会得到如下响应：

``` json
{
    "data": [
        {
            "file_name": "Duck.gif",
            "presigned_url": "https://s3.amazonaws.com/rustic-images/Duck.gif?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIARDWJNDW3U8329UDNJ%2F20200107%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20200107T050353Z&X-Amz-Expires=1800&X-Amz-Signature=1369c003b2f54510882bf9982ab56d024d6c9d2655a4d86f8907313c7499b56d&X-Amz-SignedHeaders=host",
            "tags": "animal",
            "e_tag": "\"93c570cadd6b8b2f85b47c2f14fd82a1\"",
            "is_filtered": false
        },
        {
            "file_name": "GIZMO.png",
            "presigned_url": "https://s3.amazonaws.com/rustic-images/GIZMO.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIARDWJNDW3U8329UDNJ%2F20200107%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20200107T050353Z&X-Amz-Expires=1800&X-Amz-Signature=040e76c2df5a9a54ed4fbc8490378cf732b32bae78f628448536fc610018c0c3&X-Amz-SignedHeaders=host",
            "tags": "robots",
            "e_tag": "\"2cde221a0c7a72c0a7a60cffce29a0bc\"",
            "is_filtered": false
        },
        {
            "file_name": "GreenSmile.gif",
            "presigned_url": "https://s3.amazonaws.com/rustic-images/GreenSmile.gif?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIARDWJNDW3U8329UDNJ%2F20200107%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20200107T050354Z&X-Amz-Expires=1800&X-Amz-Signature=d115b107de530ce15b3590abdbab355c2a9481a81131f88bf4ad2a59ca11bbac&X-Amz-SignedHeaders=host",
            "tags": "smile-face",
            "e_tag": "\"86854a599540f50bdc5e837d30ca34f9\"",
            "is_filtered": false
        }
    ]
}
```

前端的工作相对简单一些，我们使用的是：

- React
- React Bootstrap
- react-grid-gallery
- react-tags-input

用户可以在我们的页面浏览图片，也可以通过文件名或标签来进行检索或过滤。

![images](https://res.cloudinary.com/dxydgihag/image/upload/v1582370817/Blog/Other/learn_rust/lr-7.png)

用户还可以通过拖拽来上传文件，并且可以在提交上传之前打上标签。

![upload](https://res.cloudinary.com/dxydgihag/image/upload/v1582370815/Blog/Other/learn_rust/lr-8.png)

### 我喜欢使用Rust构建应用程序的原因

- Cargo对于依赖和应用管理的程度简直令人惊叹
- 编译器对于我们处理编译错误帮助非常大，有位博主在[博客](https://dmerej.info/blog/post/letting-the-compiler-tell-you-what-to-do/)中描述了他是如何按照编译器大指导来写代码的。我的经验也比较类似。
- 我需要的每一项功能都有crate，这让我感到非常惊喜

![Crates galore on crates.io! ](https://res.cloudinary.com/dxydgihag/image/upload/v1582370831/Blog/Other/learn_rust/lr-9.jpg)

- 在线的[Rust Playground](https://play.rust-lang.org/)，让我可以运行小的代码片段。
- Rust语言服务器，已经很好的集成到了Visual Studio Code，它能够提供实时错误检查、格式设置、符号查找等。这让我可以在几个小时内不编译就能取得不错的进展。

### 不便、惊喜和麻烦

尽管Rust的文档很棒，但我不得不依赖一些crates的文档和例子。有些crates有很棒的集成测试，提供了一些关于如何使用的提示。当然了，Stack Overflow和Reddit也给我提供了很多帮助。

![“Where’s the documentation?”](https://res.cloudinary.com/dxydgihag/image/upload/v1582370822/Blog/Other/learn_rust/lr-10.jpg)

另外还要注意的是：

- 理解所有权、生命周期和所有权借用会使学习难度陡增，特别是在为期两天的黑客马拉松中努力提供功能时。我将它们与C++做比较并且弄清楚，但有时还是会感到困惑。
- 在所有的事情中，`Strings`拦住了我几分钟，特别是`String`和`&str`的区别更是令人困惑——直到我花了些时间来理解所有权、生命周期和所有权借用才搞清楚这些。

### 其他的一些观察

- Rust中没有真正意义上的null类型，通常情况下，空值需要用`Option`类型的`None`来表示
- 模式匹配非常棒，这是我在Scala中最喜欢的一个特性，在Rust中也一样。这种代码看起来表现力很强，并且允许编译器标记未处理的情况。

``` rust
match bucket_contents {
    Err(why) => match why {
        S3ObjectError::FileWithNoName => Err(Custom(
            Status::InternalServerError,
            "Encountered bucket objects with no name".into()
        )),
        S3ObjectError::MultipleTagsWithSameName => Err(Custom(
            Status::InternalServerError,
            "Encountered a file with a more than one tag named 'tags'".into()
        ))
    },
    Ok(s3_objects) => {
        let visible_s3_objects: Vec<S3Object> = s3_objects.into_iter()
                                                          .filter(|obj| !obj.is_hidden())
                                                          .collect();
        Ok(Json(BucketContents::new(visible_s3_objects)))
    }
}
```

- 说起安全和不安全模式，你仍然可以进行更底层的编程，比如说在不安全的模式下可以和C语言代码通过接口交互。尽管Rust中有很多正确性检查，但你仍然可以在不安全模块中做一些骚操作，例如解引用。读代码的人也可以从不安全模块中获取到很多信息。
- 通过`Box`在堆中分配内存空间，而不是`new`和`delete`。刚开始感觉比较奇怪，但是也很容易理解。标准库中还定义了其他的一些[智能指针](https://doc.rust-lang.org/book/ch15-00-smart-pointers.html)，如果你需要使用引用数量或者弱引用时就可以直接使用。
- Rust中的异常也很有趣，因为它没有异常。你可以选择使用`Result<T, E>`表示可以恢复的错误，也可以用`panic!`宏表示不可恢复的错误。

``` rust
// This code:
// 1. Takes a vector of objects representing S3 contents
// 2. Uses filter to remove entries we don't care about
// 3. Uses map to transform each object into another type, but terminates iteration
// .  if the lambda passed to map returns an Err. 
// 4. If all iterations produced an Ok(S3Object) result, these are collected into a Vec<S3Object>
let bucket_contents: Result<Vec<S3Object>, S3ObjectError> = bucket_list
        .into_iter()
        .filter(|bucket_obj| bucket_obj.size.unwrap_or(0) != 0) // Eliminate folders
        .map(|bucket_obj| {
            if let None = bucket_obj.key {
                return Err(S3ObjectError::FileWithNoName);
            }

            let file_name = bucket_obj.key.unwrap();
            let e_tag = bucket_obj.e_tag.unwrap_or(String::new());
            let tag_req_output = s3_file_manager.get_tags_on_file(file_name.clone());
            let tags_with_categories: Vec<Tag> = tag_req_output.into_iter()
                                                            .filter(|tag| tag.key == "tags")
                                                            .collect();
            if tags_with_categories.len() > 1 {
                return Err(S3ObjectError::MultipleTagsWithSameName);
            }

            let tag_value = if tags_with_categories.len() == 0 {
                "".to_string()
            } else {
                tags_with_categories[0].value.clone()
            };

            let presigned_url = s3_file_manager.get_presigned_url_for_file(
                file_name.clone()
            );
            Ok(S3Object::new(
                file_name,
                e_tag,
                tag_value,
                presigned_url,
                false,
            ))
        })
        .collect();
```

手册中是这样描述的：

> 在多数情况下，Rust需要你尽可能了解错误，并且在编译之前对其做出相应的处理。这个需求使你的程序更加健壮，保证你在发布之前就可以发现并处理其中的错误。

### 要点和教训

- John Carmack曾经将编写Rust的经历描述为“非常有益”。我同意这种感受，这次hackathon给我的感觉就像是打开了一扇新世界的大门并且发现了很多新鲜事物，这些收获绝不仅仅是停留在代码层面的。
- 事后看来，我应该更加严谨的选择网络框架的。再多想一下的话，我可能会走出一条不同的道路。我下次可能会选择[iron](http://ironframework.io/)、[actix-web](https://actix.rs/), 或者是 [tiny-http](https://github.com/tiny-http/tiny-http)。
- 我只学到了Rust的皮毛，16个小时是不可能完全成为一名Rustacean的，即使我对这门语言充满了好奇心，也做了一些深入的了解。我对Rust的未来感到兴奋，我认为它为构建应用程序带来了很多规范，它是一种表现力非常丰富的语言，并且能为我们提供与C++性能相当的运行速度和内存性能呢。

### 资源

[RustIC后端代码](https://github.com/sidshank/rustic-backend)

[RustIC前端代码](https://github.com/sidshank/rustic-frontend)

[Rusoto：一个Rust的AWS SDK](https://www.rusoto.org/)

### 原文链接

https://medium.com/better-programming/learning-to-use-rust-over-a-16-hour-hackathon-5f0ac2f604df