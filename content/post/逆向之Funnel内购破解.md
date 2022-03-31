---
title: 逆向之Funnel内购破解
comments: false
toc: true
categories:
  - 逆向
tags:
  - reverse engineering
  - disassemble
date: 2017-07-05 01:02:23
---
记Funnel的内购破解。
<!-- more -->
最近学英语用[Funnel](https://itunes.apple.com/cn/app/funnel/id964666344)听国外新闻。
Funnel 包含一个 25 元的内购，解锁后可以开启自动播放和连播功能。
IAP的通用破解此处不表，Funnel多做了一步二次校验：

![](/image/IMG_2265.JPG)
提示：`Receipt verification failed`
直接搜索字符串定位代码位置：


![](/image/2017-07-05-01-13-34.jpg)

显然要绕过校验逻辑需要在两个红框处处理。
第一处汇编代码：

```
0000000100027c4c         ldrb       w25, [x0, #0x18]
0000000100027c50         tbnz       w25, 0x4, loc_100027c74

0000000100027c54         bl         imp___stubs___TMaC14SwiftyStoreKit14SwiftyStoreKit
```

tbnz的定义（ARM.Reference_Manual）：

```
TBNZ Xn|Wn, #uimm6, label

Test and Branch Not Zero: conditionally jumps to label if bit number uimm6 in register Xn is not zero. The bit number implies the width of the register, which may be written and should be disassembled as Wn if uimm is less than 32. Limited to a branch offset range of ±32KiB.
```

`w25`寄存器不为0时会触发跳转到校验逻辑块。
在偏移`0000000100027c50`处添加断点：

```
(lldb) image list -o -f | grep QUICK\ iOS\ App
[  0] 0x000000000006c000 /var/containers/Bundle/Application/2A6E51E2-ADC9-4B32-9F4E-178C0BD2E8A3/QUICK iOS App.app/QUICK iOS App(0x000000010006c000)
(lldb) br set -a 0x000000000006c000+0x0000000100027c50
Breakpoint 1: where = QUICK iOS App`_mh_execute_header + 137840, address = 0x0000000100093c50
```

第二处汇编代码：

```
0000000100027cf0         bl         imp___stubs___TZFC14SwiftyStoreKit14SwiftyStoreKit14verifyPurchasefT9productIdSS9inReceiptGVs10DictionarySSPs9AnyObject___OS_20VerifyPurchaseResult ; static SwiftyStoreKit.SwiftyStoreKit.verifyPurchase (productId : Swift.String, inReceipt : [Swift.String : Swift.AnyObject]) -> SwiftyStoreKit.VerifyPurchaseResult
0000000100027cf4         tbz        w0, 0x0, loc_100027edc

0000000100027cf8         adrp       x8, #0x100071000
```

根据`SwiftyStoreKit.SwiftyStoreKit.verifyPurchase`的值决定是否校验。
添加正则断点：

```
(lldb) rb SwiftyStoreKit.SwiftyStoreKit.verifyPurchase
Breakpoint 2: where = SwiftyStoreKit`static SwiftyStoreKit.SwiftyStoreKit.verifyPurchase (productId : Swift.String, inReceipt : Swift.Dictionary<Swift.String, Swift.AnyObject>) -> SwiftyStoreKit.VerifyPurchaseResult, address = 0x0000000100396dac
```

在Funnel中点击开启AutoPlay，触发第一个断点：

```
Process 6338 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x0000000100093c50 QUICK iOS App`_mh_execute_header + 162896
QUICK iOS App`_mh_execute_header:
->  0x100093c50 <+162896>: tbnz   w25, #0x4, 0x100093c74    ; QUICK iOS App.__TEXT.__text + 137876
    0x100093c54 <+162900>: bl     0x1000b9de0               ; symbol stub for: type metadata accessor for SwiftyStoreKit.SwiftyStoreKit
    0x100093c58 <+162904>: mov    x25, x0
    0x100093c5c <+162908>: mov    x0, x22
```

查看现有寄存器的值：

```
(lldb) p $w25
(unsigned int) $0 = 112
```

设置寄存器值为0，禁止跳转：

```
(lldb) po $w25 = 0
<nil>
```

resume线程，触发第二个断点：

```
Process 6338 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 2.1
    frame #0: 0x0000000100396dac SwiftyStoreKit`static SwiftyStoreKit.SwiftyStoreKit.verifyPurchase (productId : Swift.String, inReceipt : Swift.Dictionary<Swift.String, Swift.AnyObject>) -> SwiftyStoreKit.VerifyPurchaseResult
SwiftyStoreKit`static SwiftyStoreKit.SwiftyStoreKit.verifyPurchase (productId : Swift.String, inReceipt : Swift.Dictionary<Swift.String, Swift.AnyObject>) -> SwiftyStoreKit.VerifyPurchaseResult:
->  0x100396dac <+0>:  stp    x22, x21, [sp, #-0x30]!
    0x100396db0 <+4>:  stp    x20, x19, [sp, #0x10]
    0x100396db4 <+8>:  stp    x29, x30, [sp, #0x20]
    0x100396db8 <+12>: add    x29, sp, #0x20            ; =0x20
```

直接让线程返回0 ：

```
(lldb) thread return 0
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 2.1
    frame #0: 0x00000001000921cc QUICK iOS App`_mh_execute_header + 156108
QUICK iOS App`_mh_execute_header:
->  0x1000921cc <+156108>: mov    x0, x21
    0x1000921d0 <+156112>: bl     0x100073910               ; QUICK iOS App.__TEXT.__text + 5936
    0x1000921d4 <+156116>: mov    x0, x22
    0x1000921d8 <+156120>: bl     0x1000ba368               ; symbol stub for: swift_unknownRelease
```

成功绕过IAP校验：

![](/image/IMG_2266.PNG)

特别感谢：**《Advanced Apple Debugging & Reverse Engineering》 By Derek Selander**