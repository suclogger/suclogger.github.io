---
title: AES和RSA的JavaScript实现
comments: true
toc: true
categories:
  - 编程
tags:
  - 加密
date: 2016-09-12 01:30:28
---
<!-- abstract -->
<!-- 开始正文 -->
## 创建node工程
新建node工程，导入`crypto-js`和`crypto-browserify`的npm包。
代码如下：
```
        var CryptoJS = require('crypto-js');
        var crypto = require('crypto-browserify');

        var key = '16位密钥';
        var rsaKeyStr = ''RSA公钥';
        var rsaBuffer = new Buffer(key);
        // 使用RSA公钥加密AES的密钥
        var cryptedData = crypto.publicEncrypt(rsaKeyStr, rsaBuffer);

        //定义向量
        var iv = 'qwertyuiasdfghjk';
        key = CryptoJS.enc.Utf8.parse(key);
        iv = CryptoJS.enc.Utf8.parse(iv);
        //使用AES加密加密内容，这里用new Object 代替
        var encrypted = CryptoJS.AES.encrypt(JSON.stringify(new Object(), key, {
            iv: iv,
            mode: CryptoJS.mode.CBC,
            padding: CryptoJS.pad.Pkcs7
        });

        var encryptedObj = new Object();
        encryptedObj.mhcParams = encrypted.toString();
        encryptedObj.mhcToken = cryptedData.toString("base64");
```

注意有一个坑：
 `crypto.publicEncrypt(public_key, buffer)`传入的`public_key`参数可以是一个Object，** If public_key is a string, it is treated as the key with no passphrase and will use RSA_PKCS1_OAEP_PADDING** ：如果是个string，默认采用`RSA_PKCS1_OAEP_PADDING`的padding方式。

## 通过webpack打包
```
const name = 'SomeDemo';
const config = {
  target: 'web',
  entry: [
    './src/SomeDemo.js'
  ],
  output:{
    path: path.join(__dirname, './build/tech.suclogger.SomeDemo'),
    pathInfo: true,
    publicPath: '/build/',
    filename: name + '.js'
  },
  module: {
    loaders: [
      {
        loader: 'babel-loader',
        include: [
          path.resolve(__dirname, 'src')
        ],
        test: /\.js$/
      },
      {
        loader: 'json-loader',
        include: [
          path.resolve(__dirname, 'node_modules')
        ],
        test: /\.json$/
      }
    ]
  }
};

module.exports = config;
```

注意有一个坑：

>webpack does not natively support the loading of JSON files such as /constants-browserify/constants.json. What you should do is to use the json-loader plugin for webpack (https://www.npmjs.com/package/json-loader

webpack并不原生支持打包json格式文件，需要通过`json-loader`这个插件来打包`constants`。

## 非node环境下执行

发现报错：`Error: secure random number generation not supported by this browser`

非node环境不带`crypto`，所以缺失`crypto.getRandomValues()`方法，可以通过给打包出的js文件中添加：
```
var crypto = {
getRandomValues: function getRandomValues(array) {
var result = []
for (var i = 0, l = array.length; i < l ; i++){
result.push(Math.floor(Math.random()*256))
}

return result
}
};
```
模拟一个随机数的生成方法。
