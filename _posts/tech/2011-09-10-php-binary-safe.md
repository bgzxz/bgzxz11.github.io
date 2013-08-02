---
layout: post
title: PHP二进制安全
city: 南京
tags: [tech]
---

###前言

在PHP中经常看到一些函数有个标识「[binary safe][1]」，即二进制安全，这是个什么概念呢？ 在一个字符串中会包含很多的字符，这其中就包括NULL。「binary safe」的函数会把它的输入字符串原封不动的进行处理；而非「binary safe」的函数是在底层直接调用C的字符串相关的函数，而这些函数处理一个字符串会把NULL后边的内容忽略掉。

###例子

以下例子中，如果函数[strlen][2]是binary safe的话，我们将得到7；如果函数是非binary safe的话，我们将得到3
，由于strlen是binary safe的，所以实际上以下的运行结果是7：

	<?php
	$str = "abcx00abc"; //x00为NULL
	echo strlen($str);  //7


参考：

[http://stackoverflow.com/questions/3264514/in-php-what-does-it-mean-by-a-function-being-binary-safe](http://stackoverflow.com/questions/3264514/in-php-what-does-it-mean-by-a-function-being-binary-safe)
[http://en.wikipedia.org/wiki/Binary-safe](http://en.wikipedia.org/wiki/Binary-safe)

[1]: http://en.wikipedia.org/wiki/Binary-safe "Binary safe"
[2]: http://cn2.php.net/manual/en/function.strlen.php "strlen"