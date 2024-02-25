---
title: HTTPS之殇：绕过客户端的SSL证书绑定
comments: true
toc: true
categories:
  - 逆向
tags:
  - 安全
  - IOS
date: 2016-12-15 22:59:00
---
想必大家都听过矛和盾的故事。
之前写过一篇[文章](http://suclogger.me/%E5%BA%94%E7%94%A8%E6%8E%A5%E5%8F%A3%E7%9A%84%E5%AE%89%E5%85%A8%E6%96%B9%E6%A1%88%E8%AE%BE%E8%AE%A1%EF%BC%88%E4%BA%8C%EF%BC%89/)，谈了接口加密中用到的HTTPS加固，可以谓之盾，今天来谈谈矛。
<!-- more -->
顺带提一下最近有个消息：
>2017 年 1 月 1 日开始，苹果强制所有 iOS 应用必须使用 ATS ，即 APP 内连接必须使用更安全的 HTTPS。
[wwdc2016](https://developer.apple.com/videos/play/wwdc2016/706/)

开发者社区也是一片欢欣鼓舞。
![](/image/2016-12-15-23-09-32.jpg)

**苹果此举无疑是互联网安全的一大推动**。

但是，道高一尺魔高一丈，下面以IOS平台为例谈谈如何绕过客户端的HTTPS证书绑定。

## SSL Kill Switch
* 准备一台越狱的手机，意味着系统版本为 `IOS 9.3.3` 及以下
* 安装以下Cydia依赖：
    * Debian Packager
    * Cydia Substrate
    * PreferenceLoader
* 下载 [SSL Kill Switch2](https://github.com/nabla-c0d3/ssl-kill-switch2) 的最新Release版本：[ssl-kill-switch2 releases](https://github.com/nabla-c0d3/ssl-kill-switch2/releases) 
* 在IOS终端执行：`dpkg -i com.nablac0d3.SSLKillSwitch2_version.deb` 安装deb包
* 在IOS终端执行：`killall -HUP SpringBoard` 重启SpringBoard

完成之后，你会在手机设置中看到以下设置项：
![](/image/ SSL Kill Switch.png)
打开开关，就可以绕过客户端的SSL证书绑定了。
## How It Works
SSL Kill Switch2的工作原理是：通过`tweak`插件Hook了IOS的`Secure Transport API` 中实现SSL证书校验的相关方法。
[Secure Transport API](https://developer.apple.com/reference/security/1654508-secure_transport) 是IOS上TLS的最底层的实现，所以这个插件可以作用于IOS平台上的几乎所有应用。

实现IOS客户端证书绑定一般采用以下流程：
1. 在发起请求之前，调用`SSLSetSessionOption()`方法将`kSSLSessionOptionBreakOnServerAuth`选项设置为`true`，这个设置代表客户端要使用自己的证书校验取代系统的默认证书校验，详情见[官方文档](https://developer.apple.com/reference/security/sslsessionoption/ksslsessionoptionbreakonserverauth)；
2. 调用`SSLHandshake()`开始HTTPS握手，详情见[官方文档](https://developer.apple.com/reference/security/1400161-sslhandshake?language=objc)；
3. `SSLHandshake()`方法返回`errSSLServerAuthCompleted`后，调用`SSLCopyPeerTrust()`来获取一个连接对象，之后可以通过自定义的校验方式来校验这个对象是否合法；
4. 校验通过后调用`SSLHandshake()`来继续握手过程，或者校验失败关闭连接。

`SSL Kill Switch2`主要Hook了3个方法来绕过客户端证书校验：
###  SSLCreateContext()
通过Hook这个方法，在每次创建SSL连接时都开启自定义校验，以此禁用系统自带的证书校验：
```
static SSLContextRef replaced_SSLCreateContext (
   CFAllocatorRef alloc,
   SSLProtocolSide protocolSide,
   SSLConnectionType connectionType
) {
    SSLContextRef sslContext = original_SSLCreateContext(alloc, protocolSide, connectionType);

    // 设置 kSSLSessionOptionBreakOnServerAuth 为true来禁用系统自带的证书校验
    original_SSLSetSessionOption(sslContext, kSSLSessionOptionBreakOnServerAuth, true);
    return sslContext;
}
```
### SSLSetSessionOption()
通过Hook这个方法来过滤对`kSSLSessionOptionBreakOnServerAuth`选项的修改。
```
static OSStatus replaced_SSLSetSessionOption(
    SSLContextRef context,
    SSLSessionOption option,
    Boolean value
 ) {
    // 过滤对kSSLSessionOptionBreakOnServerAuth选项的修改
    if (option == kSSLSessionOptionBreakOnServerAuth)
        return noErr;
    else
        return original_SSLSetSessionOption(context, option, value);
}
```

### SSLHandshake()
如果`SSLHandshake()`方法返回`errSSLServerAuthCompleted`，则会触发客户端的证书校验，通过Hook这个方法，直接后续的`SSLHandshake`过程。

```
static OSStatus replaced_SSLHandshake(
    SSLContextRef context
) {
    OSStatus result = original_SSLHandshake(context);

    if (result == errSSLServerAuthCompleted) {
        // 直接继续后续的握手过程
        return original_SSLHandshake(context);
    }
    else
        return result;
}
```

## 其他
Android下也有类似的方案：[SSLUnpinning_Xposed](https://github.com/ac-pm/SSLUnpinning_Xposed)，可以绕过大部分HTTPS实现库（Java Secure Socket Extension (JSSE)，APACHE，OKHTTP）。

需要谨记的是解决问题永远没有银弹，通过HTTPS也不能完全解决应用的接口加固。
