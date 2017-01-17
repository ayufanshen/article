---  
layout: post
title: "iOS Appllication security part 2 getting class information of ios apps "
description: ""
category: ios
tags: [Security]
author: 饭小团
---   

#ios application security part 2
##getting class information of ios apps  


在此文章里，我们将分析任何app，查找里面用了哪些class，viewcontroller,内部libraries和错综复杂的细节，比如变量，函数名等。导出所有图片，plist文件，数据库等等。

##Dumping class information for Preinstalled apps on the device  
我们先拿Apple Map做实验，所有预装app都会在directory /Applications里。  
图  

![/Applications](http://highaltitudehacks.com/images/posts/ios2/20.png)  

进入 Maps.app 看看里面是啥  

![Maps app](http://highaltitudehacks.com/images/posts/ios2/21.png)  

我们看见了所有image，plist文件。可执行文件就是叫那个Maps* ，他和文件名同名    

![Maps app2](http://highaltitudehacks.com/images/posts/ios2/22.png)  

dump the class   

![dump the class](http://highaltitudehacks.com/images/posts/ios2/23.png)  

输出太多了，把他放到文件里吧，完后再用sftp下载到本地  
![dump the class2](http://highaltitudehacks.com/images/posts/ios2/24.png)  

![sftp](http://highaltitudehacks.com/images/posts/ios2/25.png)  

打开它  

![classInfo](http://highaltitudehacks.com/images/posts/ios2/26.png)  

如上。  

##Dumping class information for apps downloaded from the App store  

我的机子上的第三方app放在了这里  

	/private/var/mobile/Containers/Bundle/Application  
	以下就是app目录：  
	00605397-DF3C-4DA9-9D49-639B60F24659/ 
	96351AC2-A805-479F-B063-1753A9F74F79/
	18B657E1-8AED-46B9-9371-F4F498E60795/  	B4FCE69A-6BBE-4F07-84DD-446A05D4AA6D/
	28DB73B7-A1C5-43CE-9AE7-A0437C676BC1/  
	C109C2B7-CF08-4A59-984A-82B80A9F61F9/
	2B65F20E-5710-493C-8E49-3705B6BF9175/  
	D303B156-A870-4A25-AC50-0D38CC27DF69/
	4573A32B-AE2D-4549-8440-CD94AF535712/  
	D6B09A74-41EC-4E94-A561-77AFBFE67E36/
	588A9920-81CC-44D0-AB99-96B17428889C/  
	E4167C98-DC22-417F-9315-66520A80E13D/
	8284206A-514B-4B1A-B368-C8BAD4C96B39/  	EB5366C0-3EA2-4DF7-9078-2A73C2762A3C/
	87E22F42-F4D5-46A9-B7A7-EA2DCF15917D/  
	F59B8E8B-D054-4AFD-8365-11346DC9A264/
	8B7C7B10-7411-40AC-8F7B-44552ADA8159/  
	
从Cydia里下载clutch：  
Clutch -i 获得可以crack的app  

	iPod-touch:~ root# Clutch -i
	Installed apps:
	1:   神州专车-安全出行，必备专车软件 <com.szzc.agentcar>
	2:   今日头条-新闻、资讯、阅读、视频、话题、财经、科技、娱乐、热点、社	会、房产、直播 <com.ss.iphone.article.News>
	3:   百度地图-手机地图路线查询，智能语音交通导航 <com.baidu.map>
	4:   高德地图（专业的手机地图）-自驾、公交出行神器 志玲语音导航 	<com.autonavi.amap>
	5:   Yelp: The Best Local Food, Drinks, Services & More 	<com.yelp.yelpiphone>
	6:   汽车之家-提供买车,养车,新车报价,二手车买卖,违章查询,优惠加油等一站	式服务 <com.autohome>
	7:   支付宝 - 让生活更简单 <com.alipay.iphoneclient>
	8:   淘宝 - 随时随地，想淘就淘 <com.taobao.taobao4iphone>
	9:   Uber <com.ubercab.UberClient>
	10:  优信二手车-二手车买车卖车评估交易平台 <com.youxinpai.yxused>
	11:  一嗨租车-中国自驾用车领导品牌，出行必备 <cn.1hai.1haiiPhone>
	12:  WeChat <com.tencent.xin>
	13:  滴滴出行-专车打车出行，旅游攻略必备 <com.xiaojukeji.didi>
	14:  易到 - 低价专车,高品质出行 <com.yongche.iYongche>
	15:  高德导航-中国专业免费离线导航地图,手机汽车交通路况地铁违章查询助手 	<com.autonavi.navigation>
	16:  神州租车-中国最大租车企业，车况新，网点多，保障全！ <com.szzc.szzc>
	
随便破解一个：  
 
 	iPod-touch:~ root# Clutch -d 2
	Zipping ****.app
	ASLR slide: 0x8000
	Dumping <**** WatchKit Extension> (armv7)
	Patched cryptid (32bit segment)
	Writing new checksum
	ASLR slide: 0xa9000
	Dumping <********Extenstion> (armv7)
	Patched cryptid (32bit segment)
	Writing new checksum
	ASLR slide: 0xc1000
	Dumping <News> (armv7)
	Patched cryptid (32bit segment)
	Zipping News WatchKit Extension.appex
	Zipping NewsTodayExtenstion.appex
	Writing new checksum
	DONE: /private/var/mobile/Documents/Dumped/	com.ss.iphone.**** -iOS7.0-(Clutch-2.0.4).ipa
	Finished dumping com.ss.iphone.*** in 414.6 seconds  

文件放在这里了/private/var/mobile/Documents/Dumped/  
找见他，先解压，然后dump，按照上面的步骤传到本机上： 

	1.  unzip com.ss.iphone.***-iOS7.0-\(Clutch-2.0.4\).ipa  -d news
	2. cd /agentcar/Payload/***.app  
	3. class-dump-z ***
	
	上传
	localhost:~ fanshen$ sftp root@192.168.2.24
	root@192.168.2.24's password: 
	Permission denied, please try again.
	root@192.168.2.24's password: 
	Connected to 192.168.2.24.
	sftp> get /private/var/mobile/Documents/Dumped/***-class-dump
	Fetching /private/var/mobile/Documents/Dumped/***-class-dump to news-class-dump
	sftp>   
	
结果即可在电脑上查见。  
但有些class-dump失败，可能是对方做了保护，或者工具不匹配。这个文章里没写，我再研究研究  

文章来源：[链接](http://highaltitudehacks.com/2013/06/16/ios-application-security-part-2-getting-class-information-of-ios-apps/)
	
	





