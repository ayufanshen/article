---  
layout: post
title: "iOS Appllication security part 1 setting up a mobile pentesting platform"
description: ""
category: ios
tags: [Security]
author: 饭小团
---   


#ios application security 
#part 1 setting up a mobile pentesting platform  

##越狱 
自己越去  

###搭建审计平台  
上面你已经越狱成功了，下面我们安装一些Unix工具。  
OpenSSH:让你可以远程登录 在Cydia里搜索OpenSSH，安装  
BigBoss Recommended tools：里面有很多hack工具  
MobileTerminal：移动端命令  
现在使用你的mac登录远程登陆下你的机子，确保他们在同一网段。 查看你手机上的ip地址，假如是192.168.2.3，那么在终端里输入ssh root@192.168.2.3。默认情况下用户名密码都是alpine  

![openSSH.png](http://highaltitudehacks.com/images/posts/ios1/7.PNG)  

![BigBoss.png](http://highaltitudehacks.com/images/posts/ios1/9.PNG) 

![MobileTerminal.png](http://highaltitudehacks.com/images/posts/ios1/10.PNG)   

打开terminal，ps一下  
![ommand-ps.png](http://highaltitudehacks.com/images/posts/ios1/12.PNG)   

查看设备的ip，用ssh远程登录一下。我的是192.168.2.3，密码alpine。  
终端输入ssh root@192.168.2.3。   
![sshRoot.png](http://highaltitudehacks.com/images/posts/ios1/14.png)   

注意：在运行任何root权限的的命令时，要确保app Cydia 在后台运行。  
do an apt-get update to get the latest packages lists   
![apt-get update.png](http://highaltitudehacks.com/images/posts/ios1/15.png)  
install class-dump-z   
![class-dump-z.png](http://highaltitudehacks.com/images/posts/ios1/20x.png)  


