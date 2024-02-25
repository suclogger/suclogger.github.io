---
title: 逆向之钉钉消息和Ding已读反馈的拦截
comments: false
toc: true
categories:
  - 逆向
tags:
  - disassemble
date: 2017-06-14 23:29:29
---
有没有那么些时候，你就想假装在别处。
<!-- more -->
## 环境准备
依照用到的顺序如下：

| Tool | Usage |
| --- | --- |
| Clutch | 砸壳 |
| Hopper Disassembler v4 | 反编译，静态分析 |
| Flipboard FLEX Loader | 视图分析 |
| lldb + debugserver | 断点， 调试 |
| Flex 3 | Tweak |

首先是砸壳，细节不表，有个trick就是先使用mac端itunes下载应用同步到手机后砸壳，这样可以同时得到`arm v7`和`AArch64`版本，32位反编译后的代码可读性高，64位用于读取偏移量设置断点。

![](/image/2017-06-14-23-42-45.jpg) 

![](/image/2017-06-14-23-43-10.jpg)

## 拦截Ding已读反馈

从Ding列表点击一条未读的Ding进入这条Ding的详情，这条Ding就被标为已读，同时发送方收到已读的反馈：

![](/image/UNADJUSTEDNONRAW_thumb_1bbd.jpg?r=47)

猜测已读逻辑是放在详情页中处理，视图加载完成即执行已读相关逻辑。
通过`Flipboard FLEX Loader`查看Ding详情对应的视图：

![](/image/UNADJUSTEDNONRAW_thumb_1bbf.jpg?r=54)

可以看到对应的视图是`DTDingDetailViewControllerV2`，打开`Hopper Disassemble`中查看对应代码逻辑：

![](/image/2017-06-15-00-06-04.png)
延续之前的猜测，自然先看`viewDidAppear`方法，反编译后的代码也是一目了然：
```
void -[DTDingDetailViewControllerV2 viewDidAppear:](void * self, void * _cmd, char arg2) {
    sp = sp - 0x1c;
    r4 = self;
    loc_1c0a398(sp, @selector(viewDidAppear:), arg2);
    loc_1c0a38c(r4, @selector(dingModel));
    r6 = loc_1c0a38c(loc_1c0a390(), @selector(confirmStatus));
    sub_1c0a388();
    if (r6 == 0x0) {
            loc_1c0a38c(r4, @selector(confirmDing));
    }
    return;
}
```
大致是先判断用户的登陆状态，然后调用`confirmDing`来执行已读逻辑：
```
void -[DTDingDetailViewControllerV2 confirmDing](void * self, void * _cmd) {
    ...
    loc_1c0a38c(stack[2009], @selector(confirmDing:successBlock:failureBlock:), r8, r10, sp + 0x34, sp + 0x1c);
    ...
}
```
猜测关键逻辑在这个`confirmDing`的调用处。
`Hopper Disassemble`帮我们解析出了方法名字符串：`confirmDing`，但是没有打印出对象名，打个断点确认一下：
先回到64位确认偏移量：

![](/image/2017-06-15-13-16-40.jpg)

获取ASLR (地址空间配置随机加载) 偏移量：
```
(lldb) image list -o -f | grep DingTalk
```
 >[  0] 0x0000000000084000 /var/containers/Bundle/Application/E1551D09-5D65-485B-A283-A8E6393C4345/DingTalk.app/DingTalk(0x0000000100084000)

根据相对地址+偏移量设置断点：
```
(lldb) br set -a 0x0000000000084000+0x00000001011cb348
```
>Breakpoint 1: where = DingTalk`_mh_execute_header + 18632932, address = 0x000000010124f348

点击Ding消息进入详情，断点触发：

![](/image/2017-06-15-13-17-58.jpg)
messageSend的第一个参数就是对象了，我们通过`po`命令打印对象的类名：
```
(lldb) po [$arg1 class]
```
>DTDingServiceIMP

或者直接`po`打印实例的对象和地址：
```
(lldb) po $arg1
```
>DTDingServiceIMP: 0x174449a50

这样我们就确认了关键逻辑在`DTDingServiceIMP confirmDing`中，简单起见我们直接hook这个方法返回即可。

## 拦截消息已读反馈
跟上述大同小异，大致分析路径为：
`DTMessageOTOViewcontroller reivedMessageNotification` -> `DTMessageBaseViewController receivedMessageNotification`
-> `DTMessageControllerDataSource sendMessageReadStatusWithMessage` 

![](/image/UNADJUSTEDNONRAW_thumb_1bc2.jpg?r=47)

## Tweak
定位到对应方法之后，当然你可以通过theos环境来写一个tweak，但是有两个不好，一个是麻烦，要dump头文件，然后写xm，打包，传到手机上安装，再是不好控制开关（除非写一个开关到设置中），不然只能卸载插件才能停止hook。
更方便快捷的方式是通过`Flex 3` ：

![](/image/UNADJUSTEDNONRAW_thumb_1bc8.jpg)

我已经把Patch提交到市场，欢迎下载体验。


