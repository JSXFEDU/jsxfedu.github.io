---
title: 向浏览器推送通知
date: 2018-04-10 16:12:19
tags:
---
近几个月经常收到各个网站的通知推送。今天突发奇想，想做一个根据用户所在地理位置推送空气质量报警的服务，于是研究下。

{% asset_img 152335019648791.jpg Firefox push notification %}

看了下几个中文的介绍文档，说的都不是那么回事。通知推送有两个层级：一是基于web页面js弹出的通知，二是在页面离线时也能弹出的通知。我看的几个中文用户的文档只涉及了第一个层级。

## 使用页面JS弹出通知
这个很简单，以下两个链接解说的很清楚了：
[notification - Web API 接口 | MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/notification)
[简单了解HTML5中的Web Notification桌面通知](http://www.zhangxinxu.com/wordpress/2016/07/know-html5-web-notification/)
简单地说就是在获取用户授权之后可以拿到一个Notification对象，用它的方法就能DuangDuangDuang弹通知了。
但是这有两个问题。首要问题是，只能在页面已经在浏览器中打开的时候才能弹出通知。App对于Web的绝对优势之一就是App在没有运行的时候也能发送通知。想达到这个效果，就是第二层级做的事情了。另外一个不那么重要的问题是，如果通知是服务器发起的，你的页面JS就要和服务器保持长连接，或者去轮询服务器；这两种方式都会明显消耗服务器性能。

## 弹出离线通知
要想在用户没有打开页面的时候，服务器也能向用户发送通知的话，就要使用另外两个技术：Service worker，及其子功能PushManager。
参考：[Using the Push API](https://developer.mozilla.org/zh-CN/docs/Web/API/Push_API/Using_the_Push_API)
Demo源码下载（页面中的原始demo仓库失效了）：[push-api-demo](https://github.com/autonome/push-api-demo)
回想一下移动端是怎么做的吧。iOS的原理是，App拿到一个标记当前用户的token，然后App的服务器拿这个token向苹果的服务器发请求；iPhone和苹果的服务器保持长连接，然后App的通知就会Duang的一下出现在用户的通知中心里。如果恰巧用户正在运行这个App，那么推送的原始数据就会转由App的业务逻辑处理。
浏览器的实现也差不多。浏览器启动后，会和厂商的服务器建立长连接；JS拿到一个https地址，服务器向这个地址post数据即可将消息数据推送至浏览器。消息数据到达浏览器后并不直接显示为通知，而是由JS处理。只是这里的JS不是页面JS（页面此时可能并没有打开），而是一个预先由页面JS注册的Service worker。该worker则可以显示推送通知。按时间顺序描述就是：

1. 页面JS注册ServiceWorker
2. 页面JS在ServiceWorker的初始化成功回调中拿到PushManager，申请推送权限。
3. 用户批准推送权限后，页面JS拿到一个https推送地址，传送给服务器。
4. 服务器向用户的推送地址发送POST请求，将消息数据推到浏览器
5. ServiceWorker的push处理器被激活，Worker直接弹出通知或者将数据转交页面JS（如果页面处于打开状态）

