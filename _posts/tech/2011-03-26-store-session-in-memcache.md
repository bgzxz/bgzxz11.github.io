---
layout: post
title: PHP中用Memcache存储Session数据
city: 南京
tags: [tech]
---

###前言

[Session]是WEB程序中常用的功能，默认情况下其数据是以文件方式存储，对大访问量的场景，其处理能力较低。[Memcache]是高性能的基于内存的Key=>Value存储系统。本文将介绍如何在[PHP]中使用Memcache来存储Session数据。

###实现

在PHP中用Memcache来存储Session数据有两种方法：

1. 如果使用的是[memcache][1]扩展      
   在配置文件中进行设置：
   
		session.save_handler = memcache
		session.save_path = "tcp://127.0.0.1:11211"
	
   或者在PHP运行时进行设置：

		ini_set("session.save_handler","memcache");
		ini_set("session.save_path","tcp://127.0.0.1:11211");
	
   详见：[http://cn.php.net/manual/en/memcache.ini.php](http://cn.php.net/manual/en/memcache.ini.php)
	
2. 如果使用的是[memcached][2]扩展        
   在配置文件中进行设置：

		session.save_handler = memcached
		session.save_path = "127.0.0.1:11211"

   注意这里的`session.save_path`不需要`tcp`
   
   或者在PHP运行时进行设置：

		ini_set("session.save_handler","memcached");
		ini_set("session.save_path","127.0.0.1:11211");

   详见：[http://cn.php.net/manual/en/memcached.sessions.php](http://cn.php.net/manual/en/memcached.sessions.php)


###验证

完成设置之后，可以通过[memcachephp]来查询session数据是否被正确设置。


###参考

[http://cn.php.net/manual/en/memcache.ini.php](http://cn.php.net/manual/en/memcache.ini.php)    
[http://cn.php.net/manual/en/memcached.sessions.php](http://cn.php.net/manual/en/memcached.sessions.php)

[PHP]: http://www.php.net "PHP Hypertext Preprocessor"
[Memcache]: http://memcached.org/ "Memcache"
[Session]: http://en.wikipedia.org/wiki/Session_(computer\_science) "Session"
[memcachephp]: http://livebookmark.net/memcachephp/memcachephp.zip "memcachephp"
[1]: http://cn.php.net/manual/en/book.memcache.php "memcache extension"
[2]: http://cn.php.net/manual/en/book.memcached.php "memcached extension"
