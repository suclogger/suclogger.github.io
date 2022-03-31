---
title: 基于.Net软件的反编译初探
comments: true
toc: true
categories:
  - 逆向
tags:
  - .Net
  - de4dot
date: 2016-12-29 23:59:58
---
苦于买不到火车票，尝试了一些口碑比较好的抢票软件，顺手做了一些反编译的尝试。
<!-- more -->
成果如图，也可以看到这篇要使用的样本：

![](/image/Windows 8.1 2016-12-29 23-26-49.png)

稍微了解之后，总体感觉.Net平台下的安全相对于其他显得偏弱，常见的壳就是`.Net Reactor`和`SmartAssembly`等，而且祭出神器[de4dot](https://github.com/0xd4d/de4dot)基本可以秒所有了，如果`de4dot`还不能解决问题，那就用[`dnSpy`](https://github.com/0xd4d/dnSpy)，而且这两个出于同一个作者之手。
去了壳之后，可用的反编译软件就很多了，常用的就是[ILSpy](https://github.com/icsharpcode/ILSpy)和`Reflector`，前者开源免费，后者付费，两者都可以搭配一个强力插件[Reflexil](https://github.com/sailro/Reflexil)使用。

## 壳的检测与定性
简单的，如果直接在`Reactor`中打开看不到任何反编译出来的代码，伴随提示：
```
索引超出了数组界限。
```

![](/image/2016-12-30-00-20-05.jpg)
基本可以认定做了加壳处理。
具体壳的判定可以使用`ScanId`或者`PEiD`查看：

![](/image/2016-12-30-00-22-31.jpg)

如果无法判别具体壳的类型就比较棘手了，可能需要做手动脱壳处理，大致过程也是找偏移量dump内存。

## 砸壳
识别出壳的类型之后就可以对症下药了。这里就直接用业界最强的`de4dot`来处理。
在找一个合适的`de4dot`的时候绕了很多弯路，github上的最新代码需要自己手动编译，找了很多的版本都不尽如人意（不支持样本使用的Reactor 5代壳）。最终找到一个提供在线编译服务的站点[AppVeyor](https://ci.appveyor.com/)，总算用上了最新的`de4dot`：[de4dot](https://ci.appveyor.com/project/0xd4d/de4dot/build/artifacts)，CI果然是造福人类啊～
砸壳其实简单，难的是，砸壳不砸坏了里面的肉。就是说，砸壳拿到可以反编译的二进制文件容易，难的是不破坏软件的可执行性，`de4dot`的作者特地开了一个wiki页讲这件事[How to deobfuscate but make sure metadata tokens stay the same?](https://github.com/0xd4d/de4dot/wiki/FAQ)，主要介绍的就是如果通过选项控制来权衡反编译后代码的可读性和二进制文件的完整性。
正确的姿势是：
* 如果保留可执行性，一定要使用`--dont-rename`选项，当然如果要提高代码的可读性可以去掉
* 总是使用`-p`选项来指定软件使用的壳的类型
* 可以分别尝试32位和64位的版本

执行结果可能会有一些warning，无伤大雅：
![](/image/2016-12-30-00-42-09.jpg)
## 反编译
砸壳之后可以导入可执行文件到反编译软件中了。这里主要使用`IL Spy` + `Reflexil`。

![](/image/2016-12-30-00-46-27.png)
套路还是一样的，通过检索关键字定位方法，反编译后的代码可读性极强，`Reflexil`又极强极易用，基本几个Edit，一个Save就完成了。

- - - - --

希望明天能用这个抢到票2333333