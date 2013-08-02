---
layout: post
title: PHP_SELF，SCRIPT_NAME，SCRIPT_FILENAME，PATH_INFO，REQUEST_URI的区别
city: 南京
tags: [tech]
---

###前言

[PHP]的[$_SERVER][3]数组中存在五个和路径相关的变量：`PHP_SELF`，`SCRIPT_NAME`，      
`SCRIPT_FILENAME`，`PATH_INFO`，`REQUEST_URI`，这五个变量经常会被混淆，做下区分。

###测试环境

Nginx0.8.54 + FastCGI + PHP5.3.4

要先配置Nginx的`PATH_INFO`，在`nginx.conf`中加入如下配置：

	location ~ .* .php {
	    fastcgi_pass   127.0.0.1:9000;
	    fastcgi_index  index.php;

	    #从$fastcgi_script_name中分离出真正执行的脚本名称和PATH_INFO
	    set $real_script_name $fastcgi_script_name;

	    if ($fastcgi_script_name ~ "^(.+?.php)(/.+)$") {
	        set $real_script_name $1;
	        set $path_info $2;
	     }

	    #重新设置SCRIPT_FILENAME
	    fastcgi_param SCRIPT_FILENAME $document_root$real_script_name;
	    fastcgi_param  QUERY_STRING       $query_string;
	    fastcgi_param  REQUEST_METHOD     $request_method;
	    fastcgi_param  CONTENT_TYPE       $content_type;
	    fastcgi_param  CONTENT_LENGTH     $content_length;

	    #重新设置SCRIPT_NAME

	    fastcgi_param SCRIPT_NAME $real_script_name;
	    fastcgi_param PATH_INFO $path_info;
	    fastcgi_param  REQUEST_URI        $request_uri;
	    fastcgi_param  DOCUMENT_URI       $document_uri;
	    fastcgi_param  DOCUMENT_ROOT      $document_root;
	    fastcgi_param  SERVER_PROTOCOL    $server_protocol;
	    fastcgi_param  GATEWAY_INTERFACE  CGI/1.1;
	    fastcgi_param  SERVER_SOFTWARE    nginx/$nginx_version;
	    fastcgi_param  REMOTE_ADDR        $remote_addr;
	    fastcgi_param  REMOTE_PORT        $remote_port;
	    fastcgi_param  SERVER_ADDR        $server_addr;
	    fastcgi_param  SERVER_PORT        $server_port;
	    fastcgi_param  SERVER_NAME        $server_name;

	    # PHP only, required if PHP was built with --enable-force-cgi-redirect
	    fastcgi_param  REDIRECT_STATUS    200;
	}


我们的根目录为`/var/www`，测试域名为`example.com`（不过这个域名只能改`hosts`文件YY一下了），结构如下：

	var
	 |---www
	       |---test
	             |---test.php

###测试脚本

使用如下脚本进行测试：

	<?php
	    echo 'SCRIPT_NAME=' . $_SERVER['SCRIPT_NAME'] . '<br />';
	    echo 'SCRIPT_FILENAME=' . $_SERVER['SCRIPT_FILENAME'] . '<br />';
	    echo 'PATH_INFO=' . $_SERVER['PATH_INFO'] . '<br />';
	    echo 'REQUEST_URI=' . $_SERVER['REQUEST_URI'] . '<br />';
	?>

###测试结果

* PHP_SELF: 当前所执行的脚本的文件名，这个值是相对于根目录来说。  
   
  如果请求**http://example.com/test/test.php?k=v**，则`PHP_SELF`的值为     
  **/test/test.php**。

* SCRIPT_NAME： 当前执行的脚本的路径。    

  如果请求**http://example.com/test/test.php?k=v**，则`SCRIPT_NAME`的值     
  为**/test/test.php**。这个变量是在[CGI/1.1][1]中定义的。

* SCRIPT_FILENAME： 当前执行的脚本的绝对路径。      

  如果请求**http://example.com/test/test.php?k=v**，则`SCRIPT_FILENAME`的值     
  为**/var/www/test/test.php**。      
  
  注意：如果一个脚本以相对路径，[CLI][2]方式来执行，例如**../test/test.php**，那么     
  `$_SERVER['SCRIPT_FILENAME']`的值为相对路径，即**../test/test.php**。

* PATH_INFO：客户端提供的路径信息，即在实际执行脚本后面尾随的内容，但是会去掉**Query String**。      

  如果请求**http://example.com/test/test.php/a/b?k=v**，则`PATH_INFO`的值为**/a/b**。       
  
  [CGI1.1][1]标准中如下描述："**The PATH_INFO string is the trailing part of thecomponent of the script URI that follows the SCRIPT_NAME part of the path.**"

* REQUEST_URI：包含HTTP协议中定义的URI内容。     

  如果请求**http://example.com/test/test.php?k=v**，则`REQUEST_URI`     
  为**/test/test.php?k=v**

###区别

* PHP_SELF VS SCRIPT_NAME：
  
  `PHP_SELF`和`SCRIPT_NAME`的值在大部分情况下都是一样的，但是访问     
  **http://example.com/test/test.php/a/b?k=v**这类URL时候，`PHP_SELF`     
  为**/test/test.php/a/b**，`SCRIPT_NAME`为**/test/test.php**，可以看出`PHP_SELF`     
  比`SCRIPT_NAME`多了`PATH_INFO`的内容。

* REQUEST_URI VS SCRIPT_NAME：
  
  在访问**http://example.com/test/test.php?k=v**后，`REQUEST_URI`    
  为**/test/test.php?k=v**，`SCRIPT_NAME`为**/test/test.php**，可以看出`REQUEST_URI`     
  比`SCRIPT_NAME`多了**Query String**。
  
  如果**http://example.com/test/test.php**在服务器端做了**rewrite**： 
  
  		rewrite /test/test.php /test/test2.php; 
	
  那么`REQUEST_URI`为**/test/test.php**，`SCRIPT_NAME`为**/test/test2.php**。

###参考

[http://tools.ietf.org/html/draft-robinson-www-interface-00](http://tools.ietf.org/html/draft-robinson-www-interface-00)      
[http://stackoverflow.com/questions/279966/php-self-vs-path-info-vs-script-name-vs-request-uri](http://stackoverflow.com/questions/279966/php-self-vs-path-info-vs-script-name-vs-request-uri)   
[http://ca.php.net/manual/en/reserved.variables.server.php](http://ca.php.net/manual/en/reserved.variables.server.php)   


[PHP]: http://www.php.net "PHP Hypertext Preprocessor"
[1]: http://tools.ietf.org/html/rfc3875 "CGI/1.1"
[2]: http://php.net/manual/en/features.commandline.php "PHP CLI"
[3]: http://cn2.php.net/reserved.variables.server.php "$_SERVER"