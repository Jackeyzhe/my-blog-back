---
title: 吐血推荐珍藏的Visual Studio Code插件
date: 2019-11-11 22:46:09
tags: 技术杂谈
---

作为一名Java工程师，由于工作需要，最近一个月一直在写NodeJS，这种经历可以说是一部辛酸史了。好在有神器Visual Studio Code陪伴，让我的这段经历没有更加困难。眼看这段经历要告一段落了，今天就来给大家分享一下我常用的一些VSC的插件。<!-- more -->

VSC的插件安装方法很简单，只需要点击左侧最下方的插件栏选项，然后就可以搜索你想要的插件了。

![extends install](https://res.cloudinary.com/dxydgihag/image/upload/v1573309990/Blog/js/vsc/vs3.gif)

下面我们进入正题

### Material Theme

第一个是Material Theme，这个插件可以帮助我们修改主题色，帮助你摆脱只有黑色和白色的世界。当然你也可以通过修改setting文件来自定义主题颜色。

![Material Theme](https://res.cloudinary.com/dxydgihag/image/upload/v1573310011/Blog/js/vsc/vs4.jpg)

### Auto Import

在写Java时，通常我是直接在代码中写出类名，然后使用IDEA自动导入相应的包的，但是使用VSC时没有这个功能，这个问题就让我很困扰，这意味着作为高级crtlCV工程师，粘贴过来的代码无法直接使用，你还要去查一些引用是属于哪个包的，怎么导入。

而Auto Import帮我解决了这个大问题，它可以自动识别，解析和增加一些对应的包。有了它，我就可以继续做ctrlCV工程师了。

![Auto Import](https://res.cloudinary.com/dxydgihag/image/upload/v1573310651/Blog/js/vsc/vs5.gif)

### Import Cost

写过NodeJS的同学可能都会有一个体会，自己可能只写了几行代码，但是要安装的包竟然达到几个G，可能有些夸张，但是大量的node_modules真的很令人崩溃。

![node_modules](https://res.cloudinary.com/dxydgihag/image/upload/v1573311007/Blog/js/vsc/nj.jpg)

这时你需要的是Import Cost来帮你控制一下你导入包的大小。

![Import Cost](https://res.cloudinary.com/dxydgihag/image/upload/v1573311163/Blog/js/vsc/vs6.gif)

当你写了一个导入语句时，它会提醒你这个包的大小，如果你发现某个包太大时，就需要考虑一下你是否真的需要引入整个包了。

### Indent-Rainbow

这个插件是帮助你提升读代码的体验的，对于刚开始接触NodeJS的同学来说，读代码的时间往往比写代码的时间要多。如果项目过大时，新同学往往会迷失在很多的代码块中，分辨代码块只能靠行前缩紧数量。但是有时缩紧数量又无法一眼看出。而Indent-Rainbow就是用来帮你快速分辨代码的。

![Indent-Rainbow](https://res.cloudinary.com/dxydgihag/image/upload/v1573312918/Blog/js/vsc/vs7.png)

### Prettier — Code Formatter

Prettier插件是用来格式化代码的。

符合代码规范的代码可以说是一个工程师的脸面，而Prettier可以说是专门帮你维护脸面的插件。有了它，你在写代码时就可以肆无忌惮了，只需要在写完以后按一下对应的快捷键。你的代码就会马上变漂亮。

![Prettier](https://res.cloudinary.com/dxydgihag/image/upload/v1573313646/Blog/js/vsc/vs10.png)

### Sublime Text Keymap and Settings Importer

不知道有多少同学和我一样比较喜欢用Sublime Text。虽然ST3也非常强大，可以用来写JS代码，但是我觉得它还是比不上专业的IDE，所以我更喜欢把ST3当作「记事本」来用，如果你已经比较习惯了ST3的快捷键，并且不想因为使用VSC而改变这个习惯，那么就可以使用这个插件，它会在VSC中模仿ST3的快捷键设置。

![ST3](https://res.cloudinary.com/dxydgihag/image/upload/v1573314683/Blog/js/vsc/vs12.png)

你可以使用command+P来唤起命令窗口，然后输入`>`开始像在ST3中那样操作。

### npm Intellisense

npm Intellisense插件可以帮助你将你想要的node modules补充完整。

![npm Intellisense](https://res.cloudinary.com/dxydgihag/image/upload/v1573315069/Blog/js/vsc/vs14.gif)

### File Utils

File Utils在我看来是一个非常方（zhuang）便（bi）的插件，它可以帮助你不使用鼠标就可以创建、移动、删除文件。看起来是不是很酷。

![File Utils](https://res.cloudinary.com/dxydgihag/image/upload/v1573315346/Blog/js/vsc/vs19.gif)

### Bracket Pair Colorizer

前面我们提到了缩紧的识别，这里还有一个括号颜色标识的插件。它可以把括号标为不同的颜色，方便识别括号匹配。这种插件我在IDEA中也会用，可以极大的提高读代码的效率。

![Bracket Pair Colorizer](https://res.cloudinary.com/dxydgihag/image/upload/v1573315991/Blog/js/vsc/vs20.png)

### Trailing Spaces

这个插件会帮我们标出一些无用的尾部空格，如果发现，请立即删除它们。

![Trailing Spaces](https://res.cloudinary.com/dxydgihag/image/upload/v1573316228/Blog/js/vsc/vs25.png)

### WakaTime

这个插件很有意思，它会统计你编码的一些数据，例如各种语言的占比，日平均编码时间等。你可以用它来统计一下你每天大概的有效工作时间是多少，如果数据比较漂亮，可以不经意间让领导看到一下，哈哈哈。

![WakaTime](https://res.cloudinary.com/dxydgihag/image/upload/v1573316810/Blog/js/vsc/vs27.png)

### Vscode-icons

你是否对VSC的默认icon感到厌烦呢？你想直接通过图标看出某个文件的文件格式吗？Vscode-icons插件来帮你实现。

它会让文件的icon更加友好，也可以下载一些你喜欢的icon。

![Vscode-icons](https://res.cloudinary.com/dxydgihag/image/upload/v1573317020/Blog/js/vsc/vs32.gif)

以上就是我常用的一些VSCode的插件。喜欢的同学可以直接去市场下载体验。这些插件可能大部分都是用于提升读代码，因为我最近也是读代码比较多。如果其他同学有好用的插件也可以分享出来。

后面我也会考虑分享一些IDEA的插件，做Java的同学可以期待一波。