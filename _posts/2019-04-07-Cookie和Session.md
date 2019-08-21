---
title: Cookie和Session
date: 2019-04-07
author: 丶德灬锅
tags: Cookie Session

---

# Cookie和Session

[TOC]

> Http协议基于TCP/IP，是无状态的，如果需要保存每个客户端的请求信息，识别每个客户端的某些状态，需要使用Cookie和Session存储会话数据。
>
> 浏览器访问服务器，给服务器发送请求，访问Web应用程序，该次会话已经接通，其中不管浏览器再发送多少个请求，都视为一次会话，直到浏览器关闭本次会话结束。

## 为什么不用ServletContext、ServletRequest？

ServletContext：整个应用共享的资源，每个浏览器客户端修改会覆盖原有数据。

ServletRequest：如果使用ServletRequest，则限定了必须使用转发技术来跳转页面。

## Cookie[^1]

会话数据保存在浏览器客户端。

服务器将会话数据保存在Cookie，通过响应头发送到浏览器，浏览器端随后保存Cookie，再次请求时，如果有、Cookie信息，浏览器自动将Cookie通过请求头发送到服务器。

包含Cookie(String name, String value)，setPath(String uri)， setDomain(String domain)[^2]，setMaxAge(int expiry)，setValue(String newValue)，response.addCookie(Cookie cookie)，request.getCookies() 等接口。

Cookie默认关闭浏览器失效。

浏览器所有Cookie数量有限制，每个站点数量有限制，每个Cookie大小有限制，且只能为字符串数据，Tomcat8.5及以上支持中文Cookie，Cookie明文保存在浏览器不安全；Session可以保存非字符串数据，高并发时服务器压力大。[^3]

## Session[^1]

会话数据保存在服务器端，内存中。

服务器能识别每个不同的浏览器，每个浏览器有一个对应的Session对象保存会话数据。第一次访问时创建Session对象，并分配一个唯一的ID叫JSESSIONID，通常JSESSIONID作为Cookie值发送给浏览器，再次访问时浏览器将JSESSIONID发送到服务器，服务器通过对应的JSESSIONID查找或创建对应的Session对象。

包含getSession()，setMaxInactiveInterval(int interval)，invalidate()，getId()，setAttribute(String name, Object value)，getAttribute(String name)，removeAttribute(String name) 等接口。

Tomcat的ManagerBase类提供创建JSESSIONID的方法：随机数+时间+jvmid。

Session默认30分钟服务器自动回收，可以在web.xml中配置Session全局有效时间。

```xml
<!-- 修改session全局有效时间，单位分钟 -->
<session-config>
    <session-timeout>1</session-timeout>
</session-config>
```

避免浏览器的JSESSIONID的Cookie随浏览器关闭而丢失？手动发送一个保存到浏览器缓存目录的JSESSIONID的Cookie。

禁用Cookie后如何使用Session？通过URL重写方式，实测隐藏域提交不起作用。URL重写，手动方式如`/servlet/session;jsessionid=BAED330516608CE311B77008D0CB2267`;或者通过response.
encodeURL(String url)、response. encodeRedirectURL(String url) 两个API实现。

[^1]: [阿里云Code代码](https://code.aliyun.com/lideyu/j2ee/tree/master/src/main/java/com/ldy/servlet?accounttraceid=eff09927-c3de-439a-a0a2-0d872bdc3fdb)
[^2]: [Cookie 中的setDomain 和 setPath的区别?](https://www.jianshu.com/p/122606ffcc47) 
[^3]: [Cookie与Session的区别](https://www.jianshu.com/p/a2fe1d6441a7)