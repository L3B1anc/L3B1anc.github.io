---
layout: post
title:  "前端js校验机制绕过（burp、sqlmap可用）"
date:   2020-11-30 14：49
categories: [Pentest]
tags: Python
---
# 前端js校验机制绕过（burp、sqlmap可用）
首发https://paper.seebug.org/1415/
<!-- more -->

> 最近做银行系统比较多，遇到了很多前端校验导致无法重放也不能上扫描器和sqlmap，最后想出来了个解决办法针对js的校验可以直接绕过

最近做测试的时候，一顿测完0 high 0 medium 0 low，想着上扫描器和sqlmap一顿梭哈的时候，发现请求包一重放就失效了，这样交报告那也不能够啊，只能想想怎么绕过这个防重放机制了。

![image-20201125144745119](https://l3b1anc.oss-cn-chengdu.aliyuncs.com/uPic/XIvgHi.png)

## 发现验证机制

用burp对比了同样的两个请求，发现两个请求之间不同的只有H_TIME，H_NONCE，H_SN三个参数了，其中H_TIME一看就是时间戳。

![image-20201125145331536](https://l3b1anc.oss-cn-chengdu.aliyuncs.com/uPic/X1bUKf.png)

按照经验来说，这种类似token的值，应该是每次请求页面都会去从服务器端生成一个新的token值，通过这个token值来进行防重放的。然而，发送请求后，发现返回的包里面的参数和提交请求的参数是一样的，那这样就只剩一种情况了，就是前端通过js生成校验码发送到服务器进行校验的。

![image-20201125145821784](https://l3b1anc.oss-cn-chengdu.aliyuncs.com/uPic/SeEtmK.png)

F12大法搜搜两个关键字，发现还是某tong他老人家的安全机制，接着看看这个getUID的代码，

![image-20201125152335502](https://l3b1anc.oss-cn-chengdu.aliyuncs.com/uPic/rtOkTr.png)

```js
getUID:function(){
		var s = [];
		var hexDigits = "0123456789abcdef";
		for (var i = 0; i < 36; i++) {
			s[i] = hexDigits.substr(Math.floor(Math.random() * 0x10), 1);
		}
		s[14] = "4";  // bits 12-15 of the time_hi_and_version field to 0010
		s[19] = hexDigits.substr((s[19] & 0x3) | 0x8, 1);  // bits 6-7 of the clock_seq_hi_and_reserved to 01
		s[8] = s[13] = s[18] = s[23] = "-";

		var uuid = s.join("");
		//localStorage.setItem("LOGIN_UID",uuid);
		return uuid;
	}
```

这个代码就很简单了，生成了36位随机值，只在第15，20位有了特殊要求，那这样就随便改一位uuid就可以了，在burp上试了一下，确实是改了H_SN和H_TIME就可以了。

## 绕过脚本

但是问题来了，机制发现了，咋绕过能用扫描器扫呢，这时候想起来了原来见过大佬绕狗的操作来了，我自己写个http server做个转发把请求里面的相关参数替换掉就行了。

生成一个H_SN，刚好学习了python中execjs库可以直接执行js代码，这下方便了

```python
def uuid():
    # H_SN = 'H_SN'
    uuid1 = execjs.compile("""
    
    function uuid2(){
		var s = [];
		var hexDigits = "0123456789abcdef";
		for (var i = 0; i < 36; i++) {
			s[i] = hexDigits.substr(Math.floor(Math.random() * 0x10), 1);
		}
		s[14] = "4";  // bits 12-15 of the time_hi_and_version field to 0010
		s[19] = hexDigits.substr((s[19] & 0x3) | 0x8, 1);  // bits 6-7 of the clock_seq_hi_and_reserved to 01
		s[8] = s[13] = s[18] = s[23] = "-";

		var uuid = s.join("");
		return uuid;
	}
    """)
    H_SN = uuid1.call("uuid2")
    return(H_SN)
```

然后就是万能的flask， 自行搭建http server，然后对传入的参数自动替换掉，接着向目标网站发送替换后的请求，最后把返回数据直接扔给flask，这样无论是sqlmap还是burp的扫描，都可以直接启动了。

```python
@app.route('/<path:path>', methods=['POST'])
def post_Data(path):
    path1 = request.full_path
    data1 = json.loads(request.data)
    head = data1['head']
    head['H_SN'] = uuid()
    head['H_TIME'] = timestap1()
    data1['head'] = head
    data2 = post2bank(data1)
    return jsonify(data2), 201
```

 ## 使用效果

在repeater里面，把targert改成自己搭建的http server，向自己发送请求，通过转发替换后，可以成功绕过H_SN重放。

![image-20201125160247153](https://l3b1anc.oss-cn-chengdu.aliyuncs.com/uPic/KmcmYf.png)

最后是测试sqlmap，原数据包跑的时候一片红，根本不能用，在数据包里面把host改为127.0.0.1指向自己的http server后，还是 0 high 0 medium 0 low

![image-20201125160749877](https://l3b1anc.oss-cn-chengdu.aliyuncs.com/uPic/6fjaVc.png)

不过好歹能用了，通过这种方式，同样能够处理网站中常见的前端js加密、签名等等防爆破防重放的机制，至于为什么不用burp插件直接替换，因为是真的不会写。

![image-20201125160932668](https://l3b1anc.oss-cn-chengdu.aliyuncs.com/uPic/bbuq4D.png)

最后完整代码见https://github.com/L3B1anc/ytjs