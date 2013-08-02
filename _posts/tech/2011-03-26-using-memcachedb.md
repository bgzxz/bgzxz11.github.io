---
layout: post
title: Memcachedb的安装与使用
city: 南京
tags: [tech]
---

###前言

[Memcached]想必大家都知道,是一个缓存服务器,数据直接存在内存中，如果重启memcached进程或者重启服务器，数据就都消失了。

[Memcachedb]是一个Memcached的改进版，Memcachedb把数据存在[Berkeley DB][1]中，这样就可以使数据持久化，应用在其他的领域，[开发者][2]来自[SINA]。本文介绍下Memcachedb的安装以及使用。

###安装

1. 安装Berkeley DB 
   
   版本：Berkeley DB 4.7 or later

		$ wget http://download.oracle.com/berkeley-db/db-4.7.25.tar.gz
		$ tar -xvzf db-4.7.25.tar.gz
		$ cd db-4.7.25/
		$ cd build_unix/
		$ ../dist/configure 或者--prefix=你的路径
		$ make
		$ sudo make install

2. 安装libevent

   版本：libevent 1.3e or later（最好用1.3e）
   
		$ wget http://monkey.org/~provos/libevent-1.3e.tar.gz
		$ tar -xvzf libevent-1.3e.tar.gz
		$ cd libevent-1.3e
		$ ./configure 或者--prefix=你要安装的路径
		$ make
		$ sudo make install

    修改/etc/ld.so.conf文件，添加下面两行:

		/usr/local/lib #这个是libevent的路径
		/usr/local/BerkeleyDB.4.7/lib

    然后执行ldconfig

3. 安装memcachedb

		$ tar xvzf memcachedb-X.Y.Z.tar.gz
		$ cd memcachedb-X.Y.Z
		$ ./configure --enable-threads #-h有很多帮助
		$ make
		$ sudo make install

###使用

如下命令进行启动：

	$ /usr/local/memcachedb/bin/memcachedb -p 21201 -d -r -u \
	root -f /data1/21201.db -H /data1/demo -N -P /data1 /logs/21201.pid


###参考

[http://memcachedb.googlecode.com/svn/trunk/INSTALL](http://memcachedb.googlecode.com/svn/trunk/INSTALL)

[Memcached]: http://www.danga.com/memcached/ "Memcached"
[Memcachedb]: http://memcachedb.org/ "Memcachedb"
[1]: http://www.oracle.com/database/berkeley-db/db/index.html "berkeley db"
[2]: http://stvchu.org/ "stvchu"
[SINA]: http://sina.com "SINA"