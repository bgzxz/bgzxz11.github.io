---
layout: post
title: PHP –with-curlwrappers 导致的问题
city: 南京
tags: [tech]
---

有如下代码：

	$opts = array(
	  'http' => array(
	    'method' => "GET",
	    'header' => "Accept-language: en\r\n" . 
					"Cookie: foo=bar\r\n" . 
					"User-Agent: MyAgent/1.0\r\n"
	  )
	);

	$context = stream_context_create($opts);
	$result = file_get_contents('http://www.example.com/', false, $context);
	var_dump($result);

正常情况下以上应该输出www.example.com返回的内容，但是实际上得到的结果是空字符串，遂进行如下测试：

	$file = fopen('http://www.example.com/', 'rb');
	var_dump(stream_get_meta_data($file));

	/*
	输出结果：
	array(10) {
	  ["wrapper_data"]=>
	  array(2) {
	    ["headers"]=>
	    array(0) {
	    }
	    ["readbuf"]=>
	    resource(38) of type (stream)
	  }

	  ["wrapper_type"]=>
	  string(4) "cURL"
	  
	  ["stream_type"]=>
	  string(4) "cURL"

	  ["mode"]=>
	  string(2) "rb"

	  ["unread_bytes"]=>
	  int(0)

	  ["seekable"]=>
	  bool(false)

	  ["uri"]=>
	  string(23) "http://www.example.com/"

	  ["timed_out"]=>
	  bool(false)

	  ["blocked"]=>
	  bool(true)

	  ["eof"]=>
	  bool(false)

	}*/

输出的结果中，wrapper_type为cURL，而且wrapper\_data为空，这是不正常的，正常情况下wrapper\_type应为http，wrapper\_data数组中应该包含响应头信息。google之，发现这个现象与`--with-curlwrappers`这个编译选项有关，遂查看PHP编译参数：

	$ php -i | grep configure

	Configure Command =>  './configure' '--prefix=/usr/local/php' '--with-config-file-path=/usr/local/php/etc' '--with-mysql=/usr/local/mysql' '--with-mysqli=/usr/local/mysql/bin/mysql_config' '--with-iconv-dir=/usr/local' '--with-zlib' '--with-libxml-dir=/usr' '--enable-xml' '--enable-bcmath' '--enable-shmop' '--enable-sysvsem' '--enable-inline-optimization' '--with-curl' '--with-curlwrappers' '--enable-mbregex' '--enable-fpm' '--enable-mbstring' '--with-mcrypt' '--with-openssl' '--with-mhash' '--enable-pcntl' '--enable-sockets' '--with-xmlrpc' '--enable-zip' '--enable-soap' '--enable-bcmath'


`--with-curlwrappers`被启用了，问题应该出在这里，查看一下这个编译选项的用处：

	$ ./configure --help | grep curlwrappers
	  --with-curlwrappers     EXPERIMENTAL: Use cURL for url streams

看来`--with-curlwrappers`这个编译选项是用来处理url stream的，不过前面有个硕大的`EXPERIMENTAL`字样，还在试验中。 现有的解决方法是重新编译PHP，去掉「--with-curlwrappers」：

	$ cd /path/to/php-5.3.6
	$ make clean
	$ ./configure --prefix=/usr/local/php --with-config-file-path=/usr/local/php/etc --with-mysql=/usr/local/mysql \
	--with-mysqli=/usr/local/mysql/bin/mysql_config --with-iconv-dir=/usr/local --with-zlib \
	--with-libxml-dir=/usr --enable-xml --enable-bcmath --enable-shmop --enable-sysvsem \
	--enable-inline-optimization --with-curl --enable-mbregex --enable-fpm  --enable-mbstring --with-mcrypt \
	--with-openssl --with-mhash --enable-pcntl --enable-sockets --with-xmlrpc --enable-zip \
	--enable-soap --enable-bcmath -with-gd --with-jpeg-dir=/usr --with-png-dir=/usr --enable-gd-native-ttf

	$ make && make install
	$ cd /usr/local/php/bin
	# 删除旧的PHP binary文件，并用新的进行替换，Mac下重新编译后会产生php.dSYM文件，其他Linux系统请自行处理
	$ rm php && mv php.dSYM php 

完成后，再做个测试：

	$file = fopen('http://www.example.com/', 'rb');
	var_dump(stream_get_meta_data($file));

	/*
	结果如下：
	array(10) {
	  ["wrapper_data"]=>
	  array(12) {
	    [0]=>
	    string(18) "HTTP/1.0 302 Found"

	    [1]=>
	    string(46) "Location: http://www.iana.org/domains/example/"

	    [2]=>
	    string(13) "Server: BigIP"

	    [3]=>
	    string(17) "Connection: close"

	    [4]=>
	    string(17) "Content-Length: 0"

	    [5]=>
	    string(15) "HTTP/1.1 200 OK"

	    [6]=>
	    string(35) "Date: Sun, 18 Mar 2012 06:12:27 GMT"

	    [7]=>
	    string(29) "Server: Apache/2.2.3 (CentOS)"

	    [8]=>
	    string(44) "Last-Modified: Wed, 09 Feb 2011 17:13:15 GMT"

	    [9]=>
	    string(21) "Vary: Accept-Encoding"

	    [10]=>
	    string(17) "Connection: close"

	    [11]=>
	    string(38) "Content-Type: text/html; charset=UTF-8"

	  }

	  ["wrapper_type"]=>
	  string(4) "http"

	  ["stream_type"]=>
	  string(14) "tcp_socket/ssl"

	  ["mode"]=>
	  string(2) "rb"

	  ["unread_bytes"]=>
	  int(1225)

	  ["seekable"]=>
	  bool(false)

	  ["uri"]=>
	  string(23) "http://www.example.com/"

	  ["timed_out"]=>
	  bool(false)

	  ["blocked"]=>
	  bool(true)

	  ["eof"]=>
	  bool(false)
	}
	*/

wrapper\_type变成了http，wrapper_data也被填充了，一切恢复正常。所以一条结论：慎用`--with-curlwrappers`

参考：

[http://cn.php.net/manual/en/function.stream-context-create.php#99353](http://cn.php.net/manual/en/function.stream-context-create.php#99353)
[https://www.facebook.com/note.php?note_id=290180466652](https://www.facebook.com/note.php?note_id=290180466652)
