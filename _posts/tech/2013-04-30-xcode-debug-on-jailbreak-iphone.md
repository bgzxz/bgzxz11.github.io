---
layout: post
title: 在越狱iPhone上进行Xcode调试
city: 南京
tags: [tech]
---

![Xcode](http://{{ site.cdn }}/images/tech/xcode.jpg "Xcode")

# 说明

[苹果开发者认证](https://developer.apple.com/programs/ios/)每年`$99`，对于一个初学者和票友来说暂时没必要买，但是又想在真机上进行调试，那么只能hack了，网上相关中文资料很多，但是都有些过时，版本太老，遂总结一篇。

本文在以下环境下测试通过：

*	XCode：Version 4.6.1 (4H512) 
*	IOS SDK：6.1
*	iPhone：IOS 6.1(Jail Break)
*	Mac OS X：10.8.3 Mountain Lion

# 修改SDKSettings.plist

首先进入以下目录

	$ cd /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS6.1.sdk

在修改之前，先备份一下SDKSettings.plist：

	$ sudo cp SDKSettings.plist SDKSettings.plist.bak
	$ sudo vi SDKSettings.plist
修改`CODE_SIGNING_REQUIRED`和`ENTITLEMENTS_REQUIRED`这两个配置的值从`YES`修改为`NO`。


# 创建签名证书

首先打开`钥匙串访问`应用，然后打开`钥匙串访问->证书助理->创建证书`开始创建证书，
注意要保证证书名称为`iPhone Developer`：

![01](http://{{ site.cdn }}/images/xcode/01.png "01")

点击继续：

![02](http://{{ site.cdn }}/images/xcode/02.png "02")

添加邮箱：

![03](http://{{ site.cdn }}/images/xcode/03.png "03")

点击继续，勾选`签名`：

![04](http://{{ site.cdn }}/images/xcode/04.png "04")

点击继续，勾选`代码签名`：

![05](http://{{ site.cdn }}/images/xcode/05.png "05")

然后一路点击「继续」就可以了。

# Xcode
打开`Xcode`，选择`TARGETS`中的`Build Settings`选项卡，然后在`Code Signing`中选择我们刚创建的证书`iPhone Developer`：

![06](http://{{ site.cdn }}/images/xcode/06.png "06")

然后选择`Summary`选项卡，勾选`Use Entitlements File`：

![07](http://{{ site.cdn }}/images/xcode/07.png "07")

选中刚生成的`.entitlements`文件（不同的项目名称，生成的文件名不一样）：

![08](http://{{ site.cdn }}/images/xcode/08.png "08")

在文件中添加一个名为`get-task-allow`的配置项，类型为`Boolean`，值为`YES`：

![09](http://{{ site.cdn }}/images/xcode/09.png "09")

重启`Xcode`

# 安装Appsync

首先确保你的iPhone已经越狱，打开`Cydia`：

![10](http://{{ site.cdn }}/images/xcode/10.png "10")

然后打开`管理->编辑->添加`，添加源`http://repo.hackyouriphone.org/`：

![11](http://{{ site.cdn }}/images/xcode/11.png "11")

可能会提示源有问题，点击`仍然添加`：

![12](http://{{ site.cdn }}/images/xcode/12.png "12")

然后搜索`appsync`，安装`AppSync fo IOS 6`：

![13](http://{{ site.cdn }}/images/xcode/13.png "13")

# 调试

选择`IOS Device`进行调试：

![14](http://{{ site.cdn }}/images/xcode/14.png "14")

然后你就可以在自己的真机上调试了，Enjoy Coding　：）

# 参考

[http://stackoverflow.com/questions/15354719/code-sign-error-with-xcode4-6-on-jailbroken-device](http://stackoverflow.com/questions/15354719/code-sign-error-with-xcode4-6-on-jailbroken-device)

[http://stackoverflow.com/questions/10842595/developing-with-jailbroken-iphone-xcode](http://stackoverflow.com/questions/10842595/developing-with-jailbroken-iphone-xcode)

 