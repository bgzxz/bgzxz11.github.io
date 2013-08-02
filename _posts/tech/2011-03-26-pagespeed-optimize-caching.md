---
layout: post
title: Pagespeed优化准则之缓存优化
city: 南京
tags: [translate]
---

大多数网页中会包含很多资源，比如CSS，JavaScript，图片，这些资源一般很少有改动，所以可以缓存在浏览器。浏览器缓存可以减少HTTP的请求数量，减少请求的交互时间，减少响应的大小，减少带宽使用量。

##改进浏览器缓存(Leverage browser caching)

###概述

给资源加上一个过期时间(Expire)或者生存时间(Cache-Control: max-age)可以在浏览器上缓存那些不经常变化的资源，本地访问肯定比网络访问要快。

###具体

HTTP和HTTPS支持本地缓存，最新的浏览器(比如IE7,Chrome等)用heuristic的方式来决定对没有明确设置缓存header的资源缓存多长时间。对于老旧的浏览器，必须要对资源明确设置了缓存header才能缓存，并且有些浏览器无法缓存通过SSL加密传输的资源。为了兼容所有浏览器，还是需要对资源进行明确的缓存header设置，可缓存的资源包括图片，CSS，JavaScript，以及其他二进制文件(包 括多媒体文件，PDF，FLASH等)，HTML不需要进行缓存。

HTTP/1.1提供以下缓存用的响应头：

1. Expires 和 Cache-Control: max-age

   如果设置了这两个header之一，被请求的资源如果没有过期，那么浏览器会直接从本地读取资源而不是去服务器上获取。Cache-Control: max-age的值是相对于资源首次被访问的时间差，单位是秒，而Expires指的是资源过期的绝对日期，如果这两个header同时存 在，Cache-Control: max-age会覆盖掉Expires。

2. Last-Mofidied和ETag
   
   Last-Mofidied和ETag都是用来标示资源的有效性的，Last-Mofidied的值是个日期，ETag的值是按照规则生成的字符串。如果 访问的资源已经过期，设置了Last-Mofidied的资源会在请求中加上If-Modified-Since请求头，来询问服务器这个资源是否被修改 过，如果修改过则返回新的资源，如果没有则返回304 Not Modified。同样如果被设置了ETag的资源会在请求中加上If-None-Match请求头来询问服务器这个资源的情况，如果资源被修改过则返回 新的资源，如果没有则返回304 Not Modified。

对于网页中可以缓存的资源，Expires和Cache-Control: max-age之间需要有一个被设置，Last-Mofidied和Etag之间需要有一个被设置。这个四个header不需要都被设置，因为他们的功能用重复。

###建议

1. 给静态资源设置缓存头(Set caching headers aggressively for all static resources)

   设置Expires头的值至少一个月，最好是一年，不要超过一年，因为这个违反RFC标准。之所以选择Expires，是因为Expires比max- age的支持更加广泛。如果一个资源会发生改变，那么可以把过期时间设置的短些；如果认为一个资源会发生改变，但是又不知道什么时候，你可以设置一个比较 长的过期时间，然后可以通过URL fingerprinting来清除它。浏览器缓存不能无限设置，因为浏览器会有一个LRU算法来清除不用的缓存。设置Last-Mofidied头的 值为资源最后修改的日期。

2. 使用指纹来动态的缓存资源(Use fingerprinting to dynamically enable caching)

   如果有些资源可以被缓存，但是又更改的比较频繁，那么可以在URL中加入一个fingerprint，就是一个GET请求的参数，当资源改变需要清除浏览器缓存的时候，就改变fingerprint的值，让浏览器重新加载资源。这样的话这些资源可以设置很长的缓存时间。

3. 为IE设置正确的vary头(Set the Vary header correctly for Internet Explorer)

   IE不会缓存设置了vary头，并且其field为Accept-Encoding和User-Agent之外其他field的资源。所以要确保设置了正确的vary头。

4. 避免Firefox中URL引起的缓存冲突(Avoid URLs that cause cache collisions in Firefox)

   在Firefox中，其缓存hash算法存在问题，如果两个URL非常相近，差别不超过8个字符的话，那么它们的hash key是一样的，所以如果使用了fingerprint方式来清除缓存，那么请保证前后fingerprint的字符长度差别最少为9个字符。

5. 在Firefox中利用Cache control: public 来使HTTPS的缓存生效(Use the Cache control: public directive to enable HTTPS caching for Firefox)

   一些版本的Firefox需要在设置了普通缓存头的基础上，设置Cache-Control：public才能缓存HTTPS的资源。这个头是用来在代理服务器上缓存资源用的，但是即使设置了，代理服务器也无法缓存HTTPS的资源，所以这个调整是安全的。

##改进代理缓存(Leverage proxy caching)

###概述

对静态资源使用public缓存，让浏览器从更近的代理服务器访问资源，而不是从原始服务器访问。

###具体

HTTP除了提供浏览器缓存，还提供了代理服务器缓存机制，静态资源可以被缓存在代理服务器上，一般是ISP的服务器。即使一个用户首次访问你的网站，但 是很有可能他的资源是来自代理服务器上的缓存。使用代理缓存的益处是可以降低延迟和节省带宽。你可以通过设置Cache-control: public头来使用代理缓存。

###建议

1. 不要在静态资源中包含查询字串(Don’t include a query string in the URL for static resources)     

   大多数的代理服务器，特别明显的是squid(版本最高到3.0)，不会缓存URL中带有?的资源，即使设置了Cache-control: public，所以如果希望代理缓存服务器缓存静态资源，请把URL中的查询字串去掉，或者把这些查询参数编码，然后加在文件名中 。

2. 不要对设置cookie的资源开启代理缓存(Don’t enable proxy caching for resources that set cookies)

   设置Cache-control: public可以在不同的用户之间共享资源，但是也共享了这个资源携带的cookie。尽管大多数的代理服务器不会缓存携带cookie的资源，但是还是 应该尽可能的避免这种情况的发生。有两种方式可以选择：对携带cookie的资源设置Cache-control: private或者使用一个独立的新域名(cookieless domain)。

3. 注意JS和CSS文件的代理缓存问题(Be aware of issues with proxy caching of JS and CSS files)

   一些代理服务器不能检测到响应头中的Content-Encoding，可能会把gzip压缩过的文件发送给不支持gzip解压的浏览器，有两种方式可以 解决这个问题： 给资源设置Cache-control: private头，这样资源不会被代理服务器缓存；给资源设置Vary: Accept-Encoding头，这样代理服务器会缓存两份资源，一份为压缩过的，一份为没有压缩的，代理服务器可以根据请求头的Accept- Encoding来判断是返回哪一个资源。

原文：[http://code.google.com/speed/page-speed/docs/caching.html](http://code.google.com/speed/page-speed/docs/caching.html)
