---
title: 并发下的Base64解码问题
comments: true
toc: true
categories:
  - 编程
tags:
  - 并发
  - 加密
date: 2016-11-03 22:51:13
---
[上一篇文章](http://suclogger.tech/%E5%BA%94%E7%94%A8%E6%8E%A5%E5%8F%A3%E7%9A%84%E5%AE%89%E5%85%A8%E6%96%B9%E6%A1%88%E8%AE%BE%E8%AE%A1/)中介绍的接口加密方案上线至今已有月余，考虑到加密后影响业务的风险以及用户的升级体验，近两个版本采用了明文和密文并存的灰度升级方式，这次发布的版本中决定完全移除明文请求，所以看了一眼近一个月加密逻辑中记录的日志，发现了一些问题。
<!-- more -->
在日志中存在很多以下报错信息：
![](/image/2016-11-03-23-31-03.jpg)

![](/image/2016-11-03-23-35-33.jpg)

错误最终从RSA解密方法中抛出，错误类型有：
```
javax.crypto.BadPaddingException: data hash wrong
	at org.bouncycastle.jce.provider.JCERSACipher.engineDoFinal(Unknown Source) ~[bcprov-jdk14-138.jar:1.38.0]
	at javax.crypto.Cipher.doFinal(Cipher.java:2145) ~[na:1.7.0_10]
```
也有：
```
org.bouncycastle.crypto.DataLengthException: input too large for RSA cipher.
	at org.bouncycastle.crypto.engines.RSACoreEngine.convertInput(Unknown Source) ~[bcprov-jdk14-138.jar:1.38.0]
	at org.bouncycastle.crypto.engines.RSAEngine.processBlock(Unknown Source) ~[bcprov-jdk14-138.jar:1.38.0]
	at org.bouncycastle.crypto.encodings.OAEPEncoding.decodeBlock(Unknown Source) ~[bcprov-jdk14-138.jar:1.38.0]
	at org.bouncycastle.crypto.encodings.OAEPEncoding.processBlock(Unknown Source) ~[bcprov-jdk14-138.jar:1.38.0]
	at org.bouncycastle.jce.provider.JCERSACipher.engineDoFinal(Unknown Source) ~[bcprov-jdk14-138.jar:1.38.0]
	at javax.crypto.Cipher.doFinal(Cipher.java:2145) ~[na:1.7.0_10]
```
处理密文的切面中代码逻辑大致如下：
```
logger.info("enToken  : {}", enToken);
logger.info("enParams  : {}", enParams);
String deKey = RSAUtil.decrypt(enToken, RSAUtil.privateKey);
```
`RSAUtil`中的RSA解密逻辑为：
```
public class RSAUtil {
    private static final BASE64Decoder base64Decoder = new BASE64Decoder();
    
    public static String decrypt(String text, PrivateKey privateKey) {
        try {
            Security.addProvider(new BouncyCastleProvider());
            Cipher cipher = Cipher.getInstance("RSA/NONE/OAEPWithSHA1AndMGF1Padding", "BC");
            cipher.init(Cipher.DECRYPT_MODE, privateKey);
            byte[] encryptedData = base64Decoder.decodeBuffer(text);
            ByteArrayOutputStream out = new ByteArrayOutputStream();
            int offSet = 0;
            int i = 0;
            while (encryptedData.length - offSet > 0) {
                byte[] cache;
                if ((encryptedData.length - offSet) > (KEY_LENGTH / 8)) {
                    cache = cipher.doFinal(encryptedData, offSet, KEY_LENGTH / 8);
                } else {
                    cache = cipher.doFinal(encryptedData, offSet, encryptedData.length - offSet);
                }
                out.write(cache, 0, cache.length);
                i++;
                offSet = i * (KEY_LENGTH / 8);
            }
            return out.toString();
        } catch (Exception e) {
            LOG.error("解密发生异常", e);
            return null;
        }
    }
}
```
错误抛出的位置都在`Cipher.doFinal()`方法上。
虽然日志中报错解密失败，但是直接拿日志中打印的`enParams`和`enToken`又是可以成功解密的，这也是之所以这个报错之前没有引起我重视的原因，本地无法复现。

今天突然想到是不是因为并发的原因导致的本地无法复现，只在线上环境出现，所以写了一小段代码模拟了一下：
```
@Test
public void testdecrypt() {
    List<String> enTokenList = new ArrayList<>(100);
    for(int i=0; i<100; i++) {
        String result = RSAUtil.encrypt(String.valueOf(i), RSAUtil.publicKey);
        enTokenList.add(result);
    }
    for(String enToken : enTokenList) {
        ThreadPoolFactory.getThreadPool().execute(new TeEncrypt(enToken));
    }
}

class TeEncrypt implements Runnable{

    private String enToken;

    public TeEncrypt(String enToken) {
        this.enToken = enToken;
    }

    @Override
    public void run() {
        String deco = RSAUtil.decrypt(enToken, RSAUtil.privateKey);
        System.out.println(deco);
    }
}
```
代码的逻辑比较简单，预先生成了100条密文，然后通过线程池不加任何同步锁来模拟并发的情况，果然复现了线上的报错：
```
4
5
6
2
7
8
9
10
11
12
13
14
15
16
17
21
22
23
18
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
43
44
45
50
49
46
48
23:49:24.522 [pool-1-thread-1] ERROR tech.suclogger.common.utils.RSAUtil - 解密发生异常
javax.crypto.BadPaddingException: data hash wrong
	at org.bouncycastle.jce.provider.JCERSACipher.engineDoFinal(Unknown Source) ~[bcprov-jdk14-138.jar:1.38.0]
	at javax.crypto.Cipher.doFinal(Cipher.java:2145) ~[na:1.7.0_71]
	at tech.suclogger.common.utils.RSAUtil.decrypt(RSAUtil.java:195) [classes/:na]
	at tech.suclogger.common.utils.RSAUtil.decrypt(RSAUtil.java:220) [classes/:na]
	at tech.suclogger.web.encrypt.EncryptTest$TeEncrypt.run(EncryptTest.java:39) [test-classes/:na]
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145) [na:1.7.0_75]
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615) [na:1.7.0_75]
	at java.lang.Thread.run(Thread.java:745) [na:1.7.0_75]
52
51
53
55
56
57
58
59
60
61
null
62
23:49:24.524 [pool-1-thread-43] ERROR tech.suclogger.common.utils.RSAUtil - 解密发生异常
javax.crypto.BadPaddingException: data hash wrong
	at org.bouncycastle.jce.provider.JCERSACipher.engineDoFinal(Unknown Source) ~[bcprov-jdk14-138.jar:1.38.0]
	at javax.crypto.Cipher.doFinal(Cipher.java:2145) ~[na:1.7.0_71]
	at tech.suclogger.common.utils.RSAUtil.decrypt(RSAUtil.java:195) [classes/:na]
	at tech.suclogger.common.utils.RSAUtil.decrypt(RSAUtil.java:220) [classes/:na]
	at tech.suclogger.web.encrypt.EncryptTest$TeEncrypt.run(EncryptTest.java:39) [test-classes/:na]
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145) [na:1.7.0_75]
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615) [na:1.7.0_75]
	at java.lang.Thread.run(Thread.java:745) [na:1.7.0_75]
63
64
65
66
67
68
69
70
null
23:49:24.523 [pool-1-thread-4] ERROR tech.suclogger.common.utils.RSAUtil - 解密发生异常
javax.crypto.BadPaddingException: data hash wrong
	at org.bouncycastle.jce.provider.JCERSACipher.engineDoFinal(Unknown Source) ~[bcprov-jdk14-138.jar:1.38.0]
	at javax.crypto.Cipher.doFinal(Cipher.java:2145) ~[na:1.7.0_71]
	at tech.suclogger.common.utils.RSAUtil.decrypt(RSAUtil.java:195) [classes/:na]
	at tech.suclogger.common.utils.RSAUtil.decrypt(RSAUtil.java:220) [classes/:na]
	at tech.suclogger.web.encrypt.EncryptTest$TeEncrypt.run(EncryptTest.java:39) [test-classes/:na]
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145) [na:1.7.0_75]
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615) [na:1.7.0_75]
	at java.lang.Thread.run(Thread.java:745) [na:1.7.0_75]
null
23:49:24.523 [pool-1-thread-2] ERROR tech.suclogger.common.utils.RSAUtil - 解密发生异常
javax.crypto.BadPaddingException: data hash wrong
	at org.bouncycastle.jce.provider.JCERSACipher.engineDoFinal(Unknown Source) ~[bcprov-jdk14-138.jar:1.38.0]
	at javax.crypto.Cipher.doFinal(Cipher.java:2145) ~[na:1.7.0_71]
	at tech.suclogger.common.utils.RSAUtil.decrypt(RSAUtil.java:195) [classes/:na]
	at tech.suclogger.common.utils.RSAUtil.decrypt(RSAUtil.java:220) [classes/:na]
	at tech.suclogger.web.encrypt.EncryptTest$TeEncrypt.run(EncryptTest.java:39) [test-classes/:na]
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145) [na:1.7.0_75]
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615) [na:1.7.0_75]
	at java.lang.Thread.run(Thread.java:745) [na:1.7.0_75]
71
72
73
74
null
23:49:24.523 [pool-1-thread-42] ERROR tech.suclogger.common.utils.RSAUtil - 解密发生异常
javax.crypto.BadPaddingException: data hash wrong
	at org.bouncycastle.jce.provider.JCERSACipher.engineDoFinal(Unknown Source) ~[bcprov-jdk14-138.jar:1.38.0]
	at javax.crypto.Cipher.doFinal(Cipher.java:2145) ~[na:1.7.0_71]
	at tech.suclogger.common.utils.RSAUtil.decrypt(RSAUtil.java:195) [classes/:na]
	at tech.suclogger.common.utils.RSAUtil.decrypt(RSAUtil.java:220) [classes/:na]
	at tech.suclogger.web.encrypt.EncryptTest$TeEncrypt.run(EncryptTest.java:39) [test-classes/:na]
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145) [na:1.7.0_75]
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615) [na:1.7.0_75]
	at java.lang.Thread.run(Thread.java:745) [na:1.7.0_75]
75
76
77
78
79
80
81
82
83
null
23:49:24.526 [pool-1-thread-48] ERROR tech.suclogger.common.utils.RSAUtil - 解密发生异常
javax.crypto.BadPaddingException: data hash wrong
	at org.bouncycastle.jce.provider.JCERSACipher.engineDoFinal(Unknown Source) ~[bcprov-jdk14-138.jar:1.38.0]
	at javax.crypto.Cipher.doFinal(Cipher.java:2145) ~[na:1.7.0_71]
	at tech.suclogger.common.utils.RSAUtil.decrypt(RSAUtil.java:195) [classes/:na]
	at tech.suclogger.common.utils.RSAUtil.decrypt(RSAUtil.java:220) [classes/:na]
	at tech.suclogger.web.encrypt.EncryptTest$TeEncrypt.run(EncryptTest.java:39) [test-classes/:na]
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145) [na:1.7.0_75]
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615) [na:1.7.0_75]
	at java.lang.Thread.run(Thread.java:745) [na:1.7.0_75]
84
85
86
89
90
92
91
94
95
96
null
87
```
给异常添加断点：

![](/image/2016-11-03-23-59-11.jpg)

原因就在于，**` sun.misc.BASE64Decoder`并不是线程安全的**，并发的情况下，可以使用替代方案：
* java.util.Base64.Decoder （需要java8）
* javax.xml.bind.DatatypeConverter.parseBase64Binary(String lexicalXSDBase64Binary)
* org.apache.commons.codec.binary.Base64
...

各个方案的性能比较可以看[这篇文章](http://java-performance.info/base64-encoding-and-decoding-performance/)。

修改之后用容量为1000的线程池通过了上面的测试代码：
![](/image/2016-11-04-00-10-42.jpg)
