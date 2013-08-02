---
layout: post
title: $_REQUEST数组详解
city: 南京
tags: [tech]
---

###前言

$\_REQUEST数组是PHP中比较常用的数组，一般从其中取出POST，GET，COOKIE等参数，在这里写明一下$_REQUEST数组的填充方式，防止出现一些意想不到的问题。

###说明

在`php.ini`中有如下的配置：

	; This directive determines which super global data (G,P,C,E & S) should
	; be registered into the super global array REQUEST. If so, it also determines
	; the order in which that data is registered. The values for this directive are
	; specified in the same manner as the variables_order directive, EXCEPT one.
	; Leaving this value empty will cause PHP to use the value set in the
	; variables_order directive. It does not mean it will leave the super globals
	; array REQUEST empty.
	; Default Value: None
	; Development Value: "GP"
	; Production Value: "GP"
	; http://php.net/request-order
	
	request_order = "GP"


`request_order`这个配置项说明哪些全局变量（G，P，C，E，S分别代表`$_GET`，`$_POST`，`$COOKIE`，`$_ENV`，`$_SERVER`）的内容会被添加到`$_REQUEST`数组中，并且会指明变量填充的顺序，如果重名，那么后面填充的变量会覆盖前面填充的变量内容。如果把`request_order`置空，那么PHP将会使用`variables_order`（如下）配置项所指定的全局变量注册顺序来填充`$_REQUEST`数组，而不是说把`$_REQUEST`置空。

	; This directive determines which super global arrays are registered when PHP
	; starts up. If the register_globals directive is enabled, it also determines
	; what order variables are populated into the global space. G,P,C,E & S are
	; abbreviations for the following respective super globals: GET, POST, COOKIE,
	; ENV and SERVER. There is a performance penalty paid for the registration of
	; these arrays and because ENV is not as commonly used as the others, ENV is
	; is not recommended on productions servers. You can still get access to
	; the environment variables through getenv() should you need to.
	; Default Value: "EGPCS"
	; Development Value: "GPCS"
	; Production Value: "GPCS";
	; http://php.net/variables-order

	variables_order = "GPCS"

`variables_order`这个配置项用来指定全局变量EGPCS (Environment, Get, Post, Cookie, and Server)的解析顺序。
如果`variables_order`被设置为SP，那么PHP会创建`$_SERVER`和`$_POST`，而不会创建`$_ENV`，`$_GET`，`$_COOKIE`等变量，
如果被设置为空，那么PHP不会创建任何超级全局变量。

###注意

有时候从`$_REQUEST`中取出的值不是想要的，考虑这样一个场景：
如果在`php.ini`中设置`request_order = “GPCES”`，在HTTP请求中GET或者POST参数的name恰好与COOKIE的name相同，假如为foo。
那么在程序中通过`$_REQUEST[‘foo’]`来获取到的值是名为foo的一个cookie的值，而不是GET或者POST请求的值。

###结语

尽量不要使用`$_REQUEST`，应该从`$_GET`，`$_POST`，`$COOKIE`，`$_ENV`，`$_SERVER`等变量中取出需要的值。