---
layout: post
title:  "利用frida快速解决APP中某tong加密"
date:   2020-11-25 11：23
categories: [Android]
tags: frida
---
# 利用frida快速解决APP中某tong加密

利用frida快速解决APP中某tong加密
<!-- more -->

> 目前项目当中，遇到了客户提供的加固app使用了某tong的加解密，以前常用的xserver(https://github.com/monkeylord/XServer)对于这种加解密无法hook到传入传出了，大佬最近996也没空给我改bug了，没办法用了，最后使用了l总的HTTPDecrypt(https://github.com/lyxhh/lxhToolHTTPDecrypt/tree/master/HTTPDecrypt),也学习了一下httpdecrypt的思路，最后自行编写frida脚本快速解决app中的加密问题。

现在app加密的越来越多了，正常情况下用burp抓app的包是这样的，通过对httpdecrypt和xserver思路的学习可以通过frida快速修改body中的密文了。

![image-20201123231859812](https://i.loli.net/2020/11/25/YJixSM1v7Eea3Hr.png)

## 0x00 思路

对于现在app中的加解密，基本都使用了非对称加密，因此自行构造加解密机制的方法耗时比较久，因此这里使用了hook加解密方法的传入传出来解决加解密，在这种思路下，解决app的加解密只需要三步：

1、分析出app中的加解密方法

2、hook加解密方法

3、搭建http server，将hook到的信息发送到http server上，方便burp抓包

![image-20201123233948692](https://i.loli.net/2020/11/25/4XSKI9YMnPR23jw.png)





## 0x01 分析应用加解密方法

一开始就遇上了某加密的壳，先上万能的fdex2脱壳看看代码，结果发现是用的类抽取的壳，具体怎么脱类抽取的壳，大佬不愿意py给我，但是好在类抽取脱出来也能看见具体的类名和方法名。

![image-20201123224059044](https://i.loli.net/2020/11/25/hn7dpy4G6oCBZ3a.png)

接着将脱下来的dex文件用jadx全局搜索一下encrypt关键字，还是发现了很明显的加解密方法的，不过由于类抽取的原因，只能发现方法的传入传出，这时候没办法了就只能用l总的httpdecrypt动态调试看看。

![image-20201123224520590](https://i.loli.net/2020/11/25/mJf5O6bjpWrU29h.png)

​	具体的调试方法，httpdecrypt的github上面有很详细的说明，我就不赘述了，通过对整个类的hook和方法的遍历，找到了decryptDataWithSM方法就是加密方法了，而且第二个传入的参数就为传入的明文，那我们hook这个参数，修改它的话，就可以先于加密前修改请求了。

![image-20201123225050316](https://i.loli.net/2020/11/25/YE8DLfAv5oiPeSK.png)

## 0x02 利用frida hook加密方法

现在几家做加固的手机厂商，基本都防xposed hook了，但是对frida hook都防的不咋地，而且垃圾电脑也带不动android studio了，所以用firda方便很多，hook进加密的类后，我们考虑到要hook到请求的明文和返回包的明文，因此需要分别hook加密方法的传入和解密方法的传出，具体代码如下

```js
Java.perform(function () {
    console.log("[*] Hooking ...");
	//利用frida hook到加密的类中
    var hclass = Java.use("com.*.security.CryptoUtil");
    // hook 加密方法
    hclass.encryptDataWithSM.implementation = function (a,b,c) {
      //hook到加密方法的第一个参数
        send(arguments[1])
      //获取处理后的参数
        var op = recv(function(value) {
            console.log("[*] js recv encryptdata content: " + value);
            b =  value;
        });
        op.wait();
      //将处理后的参数返回到应用的加密方法中继续加密
        return this.encryptDataWithSM(a,b,c);
    };

    // hook 解密方法
    hclass.decryptDataWithSM.implementation = function(a,b,c){
      //获取解密后的返回值
        var getVal = this.decryptDataWithSM(a,b,c)
 			//将返回值发送到python脚本中进行处理
        send(getVal)
        var op = recv(function(value){
            console.log("[*] js recv decryptdata content: "+value);
            getVal = value;
        });
        op .wait();
        return getVal;

    };
});
```

这里介绍一下frida的send函数，通过使用send函数将js中的数据传入到python脚本中，由python脚本进行处理。这样我们就能在python脚本中获取到请求的明文和返回包的明文了。

## 0x03 将明文发送到burp中

既然都要改请求了，肯定还是要用burp才习惯吧，所以还是要把请求发送到burp中去改包，这里我选择了利用flask框架，在本地搭建了一个将接受到的请求直接返回的http服务。

```python
from flask import request, Flask, jsonify
import json

app = Flask(__name__)
app.config['JSON_AS_ASCII'] = False


@app.route('/test', methods=['POST'])
def post_Data():
    payload = json.loads(request.data)
    return jsonify(payload), 201


if __name__ == '__main__':
    app.run(debug=False, host='0.0.0.0', port=8888)
```

然后在上面的python脚本中，将获取到的明文信息直接通过requests库走burp代理，就能将明文请求发到burp上去了。

```python
def toburp(data):
    proxies = {'http':'http://127.0.0.1:8080'}
    url = 'http://127.0.0.1:8888/test'
    r=requests.post(url,data=data,proxies=proxies)
    return(r.text)
```

最后测试中的效果就像这样，请求中的就是正常通信中的密文解密后的样子，只需要在修改请求中的明文，就可以直接修改密文中对应的内容了。

![image-20201123231521709](https://i.loli.net/2020/11/25/7WzhvRBlwjYJbqi.png)

最后脱敏后的完整代码见https://github.com/L3B1anc/simpleencrypt
