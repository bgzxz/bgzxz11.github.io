---
layout: post
title: Form的enctype和action属性
city: 南京
tags: [tech]
---

###前言

[HTML]中的[form]标签想必是再熟悉不过了，其有`enctype`和`action`两个属性，平时并没有特别注意这两个属性的含义，今天介绍一下。

###说明

1. enctype
    
   form的enctype属性为编码方式，常用有两种：`application/x-www-form-urlencoded`       
   和`multipart/form-data`，默认为`application/x-www-form-urlencoded`。

2. action

   当`action`为`GET`时候，浏览器用`x-www-form-urlencoded`的编码方式把form数据转换成一个字串（name1=value1&amp; amp;name2=value2...），然后把这个字串追加到url后面，用`?`分割，然后再加载这个新的url。
   
   当`action`为`POST`时候，浏览器把form数据封装到**HTTP body**中，然后发送到服务器。
   如果没有`type=file`的`input`控件，用默认的`application/x-www-form-urlencoded`就可以了。但是如果有 `type=file`的话，就要用到`multipart/form-data`了。     
   浏览器会把整个表单以控件为单位分割，并为每个部分加上以下标识符：
   
   * `Content- Disposition`(form-data或者file)
   * `Content-Type`(默认为text/plain)
   * `name`(控件 name)等信息，
   
   并且每个标识符后面都加上分割符(**boundary**)。

如果有以下form，并选择上传文件**file1.txt**：

	<form action="http://server.com/cgi/handle" enctype="multipart/form-data" method="post">
		<input name="submit-name" type="text" value="chmod777" />
		What files are you sending?
		<input name="files" type="file" />
	</form>

则有如下body：

	Content-Type: multipart/form-data; boundary=AaB03x
	
	--AaB03x
	Content-Disposition: form-data; name=”submit-name”

	chmod777
	--AaB03x
	Content-Disposition: form-data; name=”files”; filename=”file1.txt”
	Content-Type: text/plain

	...contents of file1.txt...
	--AaB03x--

###参考

[http://www.w3.org/TR/html401/interact/forms.html#h-17.13.1](http://www.w3.org/TR/html401/interact/forms.html#h-17.13.1)


[HTML]: http://en.wikipedia.org/wiki/HTML
[form]: http://www.w3.org/TR/html401/interact/forms.html