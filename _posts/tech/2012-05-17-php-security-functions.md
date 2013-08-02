---
layout: post
title: PHP安全函数
city: 南京
tags: [tech]
---

最近读完了[《白帽子讲Web安全》][12]一书，觉得在WEB开发中安全是个非常重要的事情，但实际上工程师常抱有侥幸心理，认为这是小概率事件，但是不出事则已，一出事则非常严重，闲话少续，说说PHP中的几个跟安全相关的函数以及可用的方案。

###1. addslashes

[addslashes][1]对SQL语句中的特殊字符进行转义操作，包括(‘), (“), (\), (NUL)四个字符，此函数在DBMS没有自己的转义函数时候使用，但是如果DBMS有自己的转义函数，那么推荐使用原装函数，比如MySQL有[mysql\_real\_escape\_string][2]函数用来转义SQL。
注意在PHP5.3之前，[magic_quotes_gpc][3]是默认开启的，其主要是在$GET, $POST, $COOKIE上执行addslashes操作，所以不需要在这些变量上重复调用addslashes，否则会double escaping的。不过magic_quotes_gpc在PHP5.3就已经被废弃，从PHP5.4开始就已经被移除了，如果使用PHP最新版本可以不用担心这个问题。[stripslashes][4]为addslashes的unescape函数。

###2. htmlspecialchars

[htmlspecialchars][5]把HTML中的几个特殊字符转义成HTML Entity(格式：&amp;xxxx;)形式，包括(&amp;),('),("),(&lt;),(&gt;)五个字符。
	
	& (AND) => &amp;
	” (双引号) => &quot; (当ENT_NOQUOTES没有设置的时候)
	‘ (单引号) => &#039; (当ENT_QUOTES设置)
	< (小于号) => &lt;
	> (大于号) => &gt;	

htmlspecialchars可以用来过滤$GET，$POST，$COOKIE数据，预防[XSS][6]。注意htmlspecialchars函数只是把认为有安全隐患的HTML字符进行转义，如果想要把HTML所有可以转义的字符都进行转义的话请使用htmlentities。[htmlspecialchars_decode][7]为htmlspecialchars的decode函数。

###3. htmlentities

[htmlentities][8]把HTML中可以转义的内容转义成HTML Entity。[html_entity_decode][9]为htmlentities的decode函数。

###4. mysql_real_escape_string

[mysql_real_escape_string][10]会调用MySQL的库函数mysql_real_escape_string，对(\x00), (\n), (\r), (\), (‘), (\x1a)进行转义，即在前面添加反斜杠(\)，预防SQL注入。注意你不需要在读取数据库数据的时候调用stripslashes来进行unescape，因为这些反斜杠是在数据库执行SQL的时候添加的，当把数据写入到数据库的时候反斜杠会被移除，所以写入到数据库的内容就是原始数据，并不会在前面多了反斜杠。

###5. strip_tags

[strip_tags][4]会过滤掉NUL，HTML和PHP的标签。

###6. 结语

PHP自带的安全函数并不能完全避免XSS，推荐使用[HTML Purifier][11]


[1]: http://cn2.php.net/manual/en/function.addslashes.php "addslashes"
[2]: http://cn2.php.net/manual/en/function.mysql-real-escape-string.php "mysql-real-escape-string"
[3]: http://cn2.php.net/manual/en/security.magicquotes.php "magicquotes"
[4]: http://cn2.php.net/manual/en/function.strip-tags.php "strip-tags"
[5]: http://cn2.php.net/manual/en/function.htmlspecialchars.php "htmlspecialchars"
[6]: http://en.wikipedia.org/wiki/Cross-site_scripting "Cross site scripting"
[7]: http://cn2.php.net/manual/en/function.htmlspecialchars-decode.php "htmlspecialchars_decode"
[8]: http://cn2.php.net/manual/en/function.htmlentities.php "htmlentities"
[9]: http://cn2.php.net/manual/en/function.html-entity-decode.php "html_entity_decode"
[10]: http://cn2.php.net/manual/en/function.mysql-real-escape-string.php "mysql-real-escape-string"
[11]: http://htmlpurifier.org/ "HTML Purifier"
[12]: http://book.douban.com/subject/10546925/ "白帽子讲Web安全"