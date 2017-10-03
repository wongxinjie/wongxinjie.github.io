title: 'API: GET vs POST'
date: 2017-02-17 10:40:55
categories: Web
tags:
	- REST API
---
在开发中，开发API提供给他人用或者调用第三方API都是常用的事。而在API调用这里，发现大家对使用GET/POST都没有定则，有些API一概用GET调用，无论是安全的获取信息还是不安全的改变服务端资源状态，而有些一概用POST调用，即使只是单纯获取资源列表。那么，API调用应该使用GET还是POST，或者GET/POST混合起来用是不是更合理一点。
##### Http: GET vs POST
回到问题的本质，在考虑API是用GET还是用POST时，还是先来回顾一些HTTP协议中GET和POST的区别，下表展示GET和POST的不同之处:

| 情景   | GET                               | POST                                     |
| ---- | --------------------------------- | ---------------------------------------- |
| 刷新页面 | 没有影响                              | 会导致重复提交数据                                |
| 保存书签 | 可以被保存成书签                          | 不能保存书签                                   |
| 编码类型 | application/x-www-form-urlencoded | application/x-www-form-urlencoded或者multipart/form-data |
| 历史记录 | 参数会显示在历史记录中                       | 参数不会显示在历史记录中                             |
| 参数长度 | 参数长度有限制，服务端可能对URL有长度限制            | 没有限制                                     |
| 参数类型 | 参数只能是ASCII字符                      | 没有限制，二进制都可以                              |
| 安全问题 | 相对不安全，因为参数暴露在URL中                 | 相对安全，参数不会在历史记录中出现，也不会在服务器日志出现            |
<!--more-->

根据[RFC2616](https://www.w3.org/Protocols/rfc2616/rfc2616.html) 中HTTP方法中，HTTP方法可根据是否为安全方法(safe methods)和幂等方法(idempoten methods)分成两大类。所谓安全方法，就是请求不会对服务端产生影响，不会改变数据状态，如GET/HEAD/OPTIONS等都是safe method，而像POST/PUT/DELET会改变服务端数据，是不安全的。而幂等方法，则是N(N>1)次请求的效果和单次请求是一致的，GET/HEAD/PUT/DELETE等方法是幂等的，而POST/PATCH不是幂等的。
所以，根据HTTP协议来讲，GET作为幂等方法，应该只用来获取资源信息，而POST作为非幂等方法而且稍微安全的方法，用来改变资源信息。

##### POST比GET好？
尽管如此，可是在实际环境中，发现会有一些API还是GET一刀流，无论进行什么操作，一概都用GET方法（如twitter/Aliyun)。合理上来讲，一般用GET读用POST写。估计是想偷懒想少些一些文档。
但是还是存在一些问题：
1. 调用API通常会有一些AccessKey之类的敏感信息，这些信息暴露在URL是不是比POST不安全？
   答：实际上，POST/GET在底层并没有太大的不同，从数据传输安全角度上讲，POST并不会比GET安全，要保证真正的安全，用HTTPS。而至于这些敏感参数会保留在日志中，造成安全隐患应该不是程序讨论的问题了。
2. URL长度有现在，用POST没有这种限制？
   答：在HTTP协议中，URL是没有最大长度限制的，只是浏览器实现时对URL进行了长度现在。但在编程中URL长度可以没有现在（服务器可能会限制），但合理上来讲，进行API调用带一超大串参数不合理。

回到最初的问题，API调用用GET还是POST，合理的话按照HTTP协议来，但是根据实际情况还是各自选择。但是就个人作为POST一刀流而言，对于需要携带类似AccessKey鉴权参数的，还是用POST比较多。
最近在对接阿里云的一些对象存储/视频之类的API，在对接过程中发现加入一个API携带的参数一多，或者生成Signature的方式比较复杂，那么用GET其实有一个缺点，就是所有非ASCII字母包括那些非url-safe的字母都必须要进行quote,
而且Python标准库的urlencode依照的是RFC1738而非RFC3986，这会是一个坑。

** 参考资料 **
* [HTTP Methods: GET vs. POST](https://www.w3schools.com/tags/ref_httpmethods.asp)
* [RFC2616](https://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html#sec9)
* [Should api calls be GET or POST](http://stackoverflow.com/questions/4938276/should-api-calls-be-get-or-post)
