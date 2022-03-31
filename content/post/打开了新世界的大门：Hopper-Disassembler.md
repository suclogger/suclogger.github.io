---
title: 打开了新世界的大门：Hopper Disassembler
comments: true
toc: true
categories:
  - 逆向
tags:
  - disassemble
date: 2016-05-13 22:45:28
---
<!-- abstract -->
<!-- 开始正文 -->
在[@bleedfly](https://twitter.com/bleedfly)的影响下，也想自己动手玩一下逆向。
记录一下第一个逆向的app ： `Interface Inspector` 的过程。

使用工具：`Hopper Disassembler` Version 3.9.15 ，`Hex Fiend` Version 2.1.2 (200)
逆向APP：`Interface Inspector` Version 2.2 (17)

1. 通过检索关键字`register`定位到相关注册方法：SMLicenseManager registerLicenseWithName：
![](/image/2016-05-13 22-49-54.jpg)
2. 此处通过`verifyLicenseWithName`方法判断是否为合法注册码，定位到对应汇编代码，将变更后的汇编代码翻译为字节码。
此处通过将判断条件置反，使得无效的注册码进入有效验证码的后续逻辑，即需要将汇编代码：
```
000000010010f30e         je         0x10010f39b
```
替换为：
```
000000010010f30e         jne         0x10010f39b
```
3. 通过右侧的窗口查看汇编命令对应的字节码：

![](/image/2016-05-13 22-57-58.jpg)
根据汇编命令与字节码的对应关系：

![](/image/2016-05-13 23-00-28.jpg)

需要将`je` 对应的`0F 84 87 00 00 00 `替换为`0F 85 87 00 00 00`
4. 使用mac下的字节码编辑工具`Hex Fiend`打开MacOs目录下的可执行文件
执行替换时，因为`0F 84 87 00 00 00 `对应了多个结果，这里将后一句汇编代码对应的字节码拼接上去确保结果唯一后执行替换：
![](/image/2016-05-13 23-11-08.jpg)

5. 保存替换之后还无法运行，因为修改过的可执行文件丢失了证书信息，需要重新对该文件进行签名操作。
详细步骤可以参考：[re-sign-apples-applications](http://forums.macnn.com/79/developer-center/355720/how-re-sign-apples-applications-once/)
6. 签名之后依然无法打开，查找提示信息`Signature of the Interface Inspector is broken`找到程序启动完毕触发事件的监听部分：

![](/image/2016-05-13 23-22-40.jpg)
找到跳转入口：
![](/image/2016-05-13 23-23-31.jpg)
按照刚才的处理方法，替换此处的判断条件。
7. 至此可以打开软件并用任意字符成功注册。

![](/image/2016-05-13 23-25-18.jpg)



