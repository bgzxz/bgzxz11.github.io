---
layout: post
title: Pagespeed优化准则之减少请求开销
city: 南京
tags: [translate]
---

当客户端发送一个HTTP请求的时候，给这个域名和path设置的cookie也会随同一次发送。大多数用户的网络带宽是不对称的，上传和下载之间 的比例是1:4到1:20之间。这就是说发送一个500B的HTTP头和下载一个10KB的HTTP响应花费的时间是一样的。有的时候情况会更糟，因为 HTTP头是没有经过压缩的。在一个新的浏览器会话开始的时候，这个延迟会更大。为了避免网络拥堵，TCP会使用slow start的算法来创建新的连接。浏览器在发送新数据之前需要等待服务器返回对已经发送数据的ACK响应，这就限制了数据的发送量。如果需要发送的数据超过了一个连接一次能发送的最大量，那么数据就会分批发送，从而增加了浏览器和服务器之间交互的时间，即RTT。缩短请求时间的方法是减少请求的字节数，比如说HTTP头。

##最小化请求大小(Minimize request size)

###概述

尽可能保持cookie和请求头要小，确保一个HTTP请求可以通过一个TCP包传输完成。

###详细

理想情况下，一个HTTP请求不会用超过1个packet来传输。大多数使用范围很广的网络会限制一个packet最多能传输1500个字节；所以，如果可以把每个请求限制在1500个字节之内，那么就减少请求的开销。

HTTP请求头包括：

* Cookie: 对于需要cookie的资源，要保证cookie最小。为了使HTTP请求的数据少，对于一个域名不允许设置大于1000个字节的cookie，推荐所有cookie的平均大小小于400个字节。
* 浏览器设置的头：有些HTTP头是浏览器自动设置的，这个无法控制。
* 请求资源的URL(GET和Host字段)：拥有太多参数的URL会造成很大的请求数据量，请精简URL。
* Referrer URL

###建议

1. 用服务端存储来节省cookie带来的负载 (Use server-side storage for most of the cookie payload.)
   在cookie里存储一个唯一的标识符，然后用这个ID和服务器端的数据进行关联，一些需要存在cookie里的内容，可以在服务器端找到，从而减小了cookie的大小。

2. 删除无用或者重复的cookie字段 (Remove unused or duplicated cookie fields)
   如果cookie被设置在一个域名的顶级路径下，比如”/”，那么这个域名下所有路径都会继承这个cookie。如果想要在不同的路径中使用同一个 cookie，那么可以在顶级路径下面设置这个cookie，不需要在子路径下设置重复的cookie。如果只需要在子路径中使用一个cookie，那么 不要在顶级路径中设置这个cookie，因为这样会导致cookie会发送到不需要此cookie的路径中，造成不必要的流量。

##使用cookie无关的域名来传输静态内容 (Serve static content from a cookieless domain)

###概述

使用一个cookie无关的域名来传输静态资源，可以减少页面总体的流量。

###详细

静态的内容，例如图片，JS，CSS等，不需要传递cookie，因为没有用户会与这些资源进行交互。可以使用cookie无关的域名来传输静态资源，这 样可以降低请求的延迟。这个技术对于其中包含大量很难被缓存的内容的页面特别有效，比如经常修改的缩略图或者不经常被访问的图片。对于页面中包含超过5个静态资源的页面推荐使用这种技术，如果少于5个，那么不值得这么做。如果想使用cookie无关的域名来传输静态资源，需要注册一个域名，然后添加 CNAME记录，并指向到已经存在的A记录上。调整程序，使所有的静态资源都通过cookie无关的域名进行传输。

###建议

1. 开启代理缓存 (Enable proxy caching)
   对于很难改变的内容，设置浏览器和代理缓存。由于这些资源不会附带cookie，所以不用担心代理服务器会缓存用户的数据。

2. 不要用cookie无关的域名来传输外部JS (Don’t serve early loaded external JS files from the cookieless domain)
   在HTML文档头部引入并且页面渲染所需要的外部JavaScript文件，应该使用和页面一样的域名来传输，而不是使用cookie无关的新域名，因为 浏览器会阻塞JavaScript后面其他的资源下载，直到所有的JavaScript完成下载，解析，执行。所以如果使用cookie无关的域名则可能 会增加额外的DNS查询的时间。

原文：[http://code.google.com/speed/page-speed/docs/request.html](http://code.google.com/speed/page-speed/docs/request.html)

