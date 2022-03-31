---
title: 逆向解除TextNut内购限制
comments: true
toc: true
categories:
  - 逆向
tags:
  - disassemble
date: 2016-05-15 11:41:07
---
看到少数派的[微博](http://weibo.com/sspaime)上推荐了一款markdown编辑器，体验了一下觉得不错，但是有未内购只能创建2个文件夹的限制，遂尝试通过逆向解除限制。
<!-- more -->

>半年前写这篇文章的时候，因为软件作者的抗议和谐了文章内容，现在将内容放出。 -20161216

使用工具：Hopper Disassembler，
逆向App：TextNut

首先看一下`applicationWillFinishLaunching`方法，一般这里会有注册状态的相关逻辑：
```
if ([rbx areYouSponsor] == 0x1) {
                            r12 = [[ESPurchaseManager sharedPurchaseManager] retain];
                            rbx = [[SKPaymentQueue defaultQueue] retain];
                            rdx = r12;
                            [rbx addTransactionObserver:rdx];
                            rdi = rbx;
                            [rdi release];
                            [r12 release];
                    }
```

猜测是用`areYouSponsor`来存放注册状态，分析这个变量的相关引用：
![](/image/2016-05-15 12-16-53.jpg)
可以看到如果注册成功，`areYouSponsor`会被设置为`0x1`，继续看这个变量对创建文件夹的控制：
在`ETDocumentController validateMenuItem`方法中：
```
if ([r12->settings areYouSponsor] == 0x0) {
                    if ((_objc_msgSend(r13, rbx) == 0x1772) && ([ESCoreDataUtil countLibraries:0x1] > 0x3)) {
                            r15 = 0x0;
                    }
                    else {
                            if (_objc_msgSend(r13, rbx) == 0x1776) {
                                    if ([ESCoreDataUtil countLibraries:0xa] <= 0x1) {
                                            r15 = 0x1;
                                    }
                                    else {
                                            r15 = 0x0;
                                    }
                            }
                            else {
                                    r15 = 0x1;
                            }
                    }
            }
            else {
                    r15 = 0x1;
            }
```
这段代码通过`countLibraries`来计算当前library数量和判断`areYouSponsor`来控制新建Library菜单的是否只读。
尝试将判断条件置反，即将`if ([r12->settings areYouSponsor] == 0x0) {`跳转的逻辑`和else {`跳转的逻辑对调：
![](/image/2016-05-15 12-30-03.jpg)
将`7566`改为`7466`：
![](/image/2016-05-15 12-31-20.jpg?r=35)
解除限制成功：
![](/image/2016-05-15 12-32-37.jpg)


