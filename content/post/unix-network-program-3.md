---
title: UNIX网络编程学习笔记（三）
comments: true
toc: true
categories:
  - 编程
tags:
  - UNIX
date: 2020-05-17 10:30:31
---
UNIX网络编程学习笔记（三）
<!-- more -->

## 大纲
![](/image/2020-05-17/UNIX网络编程 第三章.png)

## 习题
### “3.1　为什么诸如套接字地址结构的长度之类的值-结果参数要用指针来传递？”
之所以需要通过指针来传递，是因为这个参数同时扮演了两个角色：入参和出参。
作为入参，应用程序告诉内核需要操作的空间大小，避免写越界。
作为出参，内核高速应用程序实际写入的空间大小，从而控制应用程序行为。
> 关于值-结果的解释可以再参考： [网络编程中为什么使用“值-结果”参数_chuanglan的专栏-CSDN博客](https://blog.csdn.net/chuanglan/article/details/80679709)

### “3.2　为什么readn和writen函数都将void *型指针转换为char *型指针？”
首先需要思考，指针仅仅是地址而已，为什么需要有类型？这里主要有两个考虑：
1. 不同的数据占用的内存单元数十不一样的！也就是int*指向的数据和float*指向的数据占用的内存单元不一样，那么我们通过指针来取数据的时候，需要能知道读取地址后的几个单元
2. 当指针在移动的时候我们通过类型就知道一步移动要涉及几个内存单元
如果不转换为`char*`指针，是无法进行上述的相关运算的。

那为什么不直接在函数声明中定义为`char *`型？
个人理解这个是接口声明的艺术。如果直接定义为`char *`型，就提高了函数使用和理解的成本。具体使用什么指针交由运算时再确认是最优雅的。

>关于这段解释可以再参考：[指针和指针强制转换( 回忆版 )-------让初学者理解_c/c++_shanshanpt的专栏-CSDN博客](https://blog.csdn.net/shanshanpt/article/details/8543955?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase)
















