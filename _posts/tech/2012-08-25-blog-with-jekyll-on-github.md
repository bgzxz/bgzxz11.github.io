---
layout: post
title: 利用Jekyll在Github上写博客
city: 南京
tags: [tech]
---

之前个人博客是托管在[Tumblr][1]上的，不过最近Tumblr被墙了，除了问候方校长之外，一个随之而来的事情就是要迁移博客。WordPress肯定不会再用了，太臃肿，这也是我放弃WordPress使用Tumblr的原因，所以选择下一个博客平台一定要轻量级。偶然之间看到BeiYuu的这篇[文章][2]，发现了[Jekyll][3]这个静态模板系统，并且可以把博客内容托管在[Github][4]上，这对于程序员来说是太适合不过的了，闲言少叙，开始动手！

首先在本地上安装Jekyll，本人使用的系统是Mac OS X Mountain Lion，并且已经安装好Ruby，Python，请读者根据自身情况自行安装。
	
	$ gem install jekyll

安装RDiscount

	$ gem install rdiscount

安装pygments
	
	$ easy_install pip
	$ pip install --upgrade distribute
	$ pip install pygments

安装完成之后，clone [Tom Preston-Werner][5]做的模板，并删除\_posts，\_images两个目录中的内容，因为里边的内容是有版权的。完成之后，在_posts中添加自己的文章，格式可以markdown或者Textile。
	
	$ git clone git://github.com/mojombo/mojombo.github.com.git myblog
		
启动jekyll创建网站静态内容，然后你可以通过127.0.0.1:4000来访问自己的博客
	
	$ jekyll --server
	[2012-08-25 15:37:04] INFO  WEBrick 1.3.1
	[2012-08-25 15:37:04] INFO  ruby 1.9.3 (2012-04-20) [x86_64-darwin12.0.0]
	[2012-08-25 15:37:04] INFO  WEBrick::HTTPServer#start: pid=2687 port=4000

如果以上没有问题，接下来就要进行数据[迁移][7]，支持WordPress，Drupal，Movable Type，Typo 4+，TextPattern，Mephisto，Blogspot，Posterous，Tumblr，我之前使用的是Tumblr，所以进行如下操作
	
	$ gem install sequel mysqlplus json
	$ pip install html2text
	
在项目下创建_import目录，然后执行迁移操作

	$ cd  /Users/weizhifeng/dev/jekyll/jeremywei.github.com && mkdir _import
	$ ruby -rubygems -e 'require "jekyll/migrators/tumblr"; Jekyll::Tumblr.process("http://www.your_blog_url.com", format="md")'

数据迁移完成之后，需要在[Github][4]上创建项目username.github.com「username是你的Github用户名」，然后把博客的origin替换为新建的项目，并且push到Github，十分钟后就可以通过http://username.github.com来访问了。
	
	$ cd myblog
	$ git remote rm origin
	$ git remote add origin git@github.com:username/username.github.com.git
	$ git push origin 
	
如果你有独立域名，那么你可以把域名[指向][8]到Github，首先创建名为CNAME的文本文件，内容为域名
	
	$ touch CNAME && echo "example.com" > CNAME
	
接下来在域名example.com中添加A记录指向204.232.175.78

[1]: http://www.tumblr.com "Tumblr"
[2]: http://beiyuu.com/github-pages/ "github pages"
[3]: https://github.com/mojombo/jekyll "jekyll"
[4]: http://github.com "github"
[5]: https://github.com/mojombo/mojombo.github.com "Tom Preston-Werner"
[6]: https://github.com/mojombo "mojombo"
[7]: https://github.com/mojombo/jekyll/wiki/Blog-Migrations "migrations"
[8]: https://help.github.com/articles/setting-up-a-custom-domain-with-pages "custom domain"
