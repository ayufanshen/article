---  
layout: post
title: "iOS application security part 10 ios filesystem and forensics"
description: ""
category: ios
tags: [Security]
author: 饭小团
---   


#ios application security part 10 
##ios filesystem and forensics  

本章，我们继续查看iOS 文件系统，理解目录的组织，查看重要的文件。如何从plist和数据库里拿到数据。我们也会app如何把数据存到特别的目录里并且我们该如何抽取他们。  

在之前的文章里，其中一个最重要的事就是。我们登录到设备里的都是root，而其他都是mobile级别。mobile用户只有很少的权限。所有app都是mobile级别，除了Cydia。还有一些apple内部的守护进程和服务是root级别。ps aux可以看的很清楚。默认情况下，一旦你设备越狱，root和mobile的密码都是alpine  

![ps aux](http://highaltitudehacks.com/images/posts/ios10/1.png)  

It is possible for you to configure an app to run with root privileges. For more details on it, check out this answer on Stack Overflow.  

ssh进设备，去/Applications.文件。可以看见预装的app和一些通过Cydia安装的。比如Terminal app。但这些都没运行在沙盒内，而/var/mobile/Applications里的才是，他们都默认为mobile模式，除非经过特殊配置才会运行在root模式下。不过沙盒内容以后再讨论  
cd /Applications 里面都是预装app  
![](http://highaltitudehacks.com/images/posts/ios10/2.png)  

从appStore里下载的app会装在这里/private/var/mobile/Containers/Bundle/Application ，还有你从Cydia和自己安装的ipa，也会放在这里。  

	fanshende-iPod-touch:/private/var/mobile/Containers/Bundle/Application root# ls -l
	total 0
	drwxr-xr-x 3 mobile mobile 204 May 27  2016 00605397-DF3C-4DA9-9D49-639B60F24659/
	drwxr-xr-x 3 mobile mobile 204 Oct  7 11:07 18B657E1-8AED-46B9-9371-F4F498E60795/
	drwxr-xr-x 3 mobile mobile 204 Dec 27 14:10 28DB73B7-A1C5-43CE-9AE7-A0437C676BC1/
	drwxr-xr-x 3 mobile mobile 204 Oct  7 11:07 2B65F20E-5710-493C-8E49-3705B6BF9175/
	drwxr-xr-x 3 mobile mobile 204 Jan  6 17:56 3BB79FD4-C237-4A74-8AF0-3D0608D92428/
	drwxr-xr-x 3 mobile mobile 204 Jun  6  2016 4573A32B-AE2D-4549-8440-CD94AF535712/
	drwxr-xr-x 3 mobile mobile 204 Jan  6 17:53 4A2F60D1-B12C-4344-944F-F79A311CBB3B/
	drwxr-xr-x 3 mobile mobile 136 Oct  5 05:53 588A9920-81CC-44D0-AB99-96B17428889C/
	drwxr-xr-x 3 mobile mobile 238 Jun  6  2016 8284206A-514B-4B1A-B368-C8BAD4C96B39/
	drwxr-xr-x 3 mobile mobile 238 Oct  7 11:06 87E22F42-F4D5-46A9-B7A7-EA2DCF15917D/
	drwxr-xr-x 3 mobile mobile 204 Oct  7 11:10 8B7C7B10-7411-40AC-8F7B-44552ADA8159/
	drwxr-xr-x 3 mobile mobile 204 Jun  6  2016 96351AC2-A805-479F-B063-1753A9F74F79/
	drwxr-xr-x 3 mobile mobile 204 Jan 10 11:31 A3EBCEF7-D99C-4417-9813-E6CDF5B8DA75/
	drwxr-xr-x 3 mobile mobile 238 Jun  6  2016 B4FCE69A-6BBE-4F07-84DD-446A05D4AA6D/
	drwxr-xr-x 3 mobile mobile 204 Oct  7 11:07 C109C2B7-CF08-4A59-984A-82B80A9F61F9/
	drwxr-xr-x 3 mobile mobile 204 Jan  6 17:57 D5DB8A3D-AEAD-4DF8-9816-BAA9A9FA4420/
	drwxr-xr-x 3 mobile mobile 204 Jun  6  2016 E4167C98-DC22-417F-9315-66520A80E13D/
	drwxr-xr-x 3 mobile mobile 204 Oct  7 11:09 F59B8E8B-D054-4AFD-8365-11346DC9A264/

随便打开一个看看，这个是WeChat： 
	
	iPod-touch:/private/var/mobile/Containers/Bundle/Application/4A2F60D1-B12C-4344-944F-F79A311CBB3B/WeChat.app root# ls
	AlbumCommentPlainTextTip\@2x.jpg  Expression_91\@2x.png                  fav_fileicon_number90\@3x.png
	AlbumListViewBkg\@2x.jpg          Expression_92\@2x.png                  fav_fileicon_page90\@2x.png
	AppIcon60x60\@2x.png              Expression_93\@2x.png                  fav_fileicon_page90\@3x.png
	AppIcon60x60\@3x.png              Expression_94\@2x.png                  fav_fileicon_pdf90\@2x.png
	AppIcon76x76\@2x~ipad.png         Expression_95\@2x.png                  fav_fileicon_pdf90\@3x.png
	AppIcon76x76~ipad.png             Expression_96\@2x.png                  fav_fileicon_pic90\@2x.png
	AppIcon83.5x83.5\@2x~ipad.png     Expression_97\@2x.png                  fav_fileicon_pic90\@3x.png
	AppVoiceAssistant\@2x.jpg         Expression_98\@2x.png                  fav_fileicon_ppt90\@2x.png
	Assets.car                        Expression_99\@2x.png                  fav_fileicon_ppt90\@3x.png
	ChannelInfo.plist                 Expression_9\@2x.png                   fav_fileicon_recording90\@2x.png
	ChatBackgroundThumb_00\@2x.jpg    FaceAuth.xml                           fav_fileicon_recording90\@3x.png
	EmoitconImageLoading.gif          Fav_list_error\@2x.png                 fav_fileicon_txt90\@2x.png
	Emoticon.xml                      Fav_list_error\@3x.png                 fav_fileicon_txt90\@3x.png
	Emoticon_tusiji_banner.jpg        GTMOAuth2ViewTouch.nib                 fav_fileicon_unknow90\@3x.png
	Entitlements_for_appstore.plist   GlobalSearch.xml                       fav_fileicon_unkown90\@2x.png
	Entitlements_for_ext.plist        Icon\@2x.png                           fav_fileicon_video90\@2x.png
	Entitlements_for_jailbreak.plist  Info.plist                             fav_fileicon_video90\@3x.png
	Entitlements_wc.plist             JSB.pic                                fav_fileicon_voice90\@2x.png
	Entitlements_wc_for_ext.plist     JSB_B.pic                              fav_fileicon_voice90\@3x.png
	Entitlements_wx.plist             JSB_J.pic                              fav_fileicon_word90\@2x.png
	Entitlements_wx_for_ext.plist     JSB_S.pic                              fav_fileicon_word90\@3x.png
	Expression_100\@2x.png            JSPatch.js                             fav_fileicon_xls90\@2x.png
	Expression_101\@2x.png            KitSetting.xml                         fav_fileicon_xls90\@3x.png
	Expression_102\@2x.png            LBSErrorViewController.nib             fav_fileicon_zip90\@2x.png
	Expression_103\@2x.png            Launch\ Screen.storyboardc/            fav_fileicon_zip90\@3x.png
	Expression_104\@2x.png            LaunchImage-700-568h\@2x.png           fr.lproj/
	Expression_105\@2x.png            LaunchImage-700-Portrait\@2x~ipad.png  gallery.ckcomplication/
	Expression_10\@2x.png             LaunchImage-700\@2x.png                he.lproj/
	Expression_11\@2x.png             LaunchImage\@2x.png                    hi.lproj/
	Expression_12\@2x.png             Localizable.strings                    id.lproj/
	Expression_13\@2x.png             M03C_150519.wav                        image.css
	Expression_14\@2x.png             ModernStyle.xml                        in.caf
	Expression_15\@2x.png             Monaco.ttf                             ios-chs_1.jpg
	Expression_16\@2x.png             PayCardPassword.png                    ios-en_1.jpg
	Expression_17\@2x.png             PayCardPassword\@2x.png                ios6-chs_2.jpg

##Gathering information from database files  

apple使用sqlite作为数据库，文件结尾都是.db或.sqlitedb。很多功能Core Data, NSUserDefaults等底层都是sqlite。我们很容易从操作系统或某些app里获取这些数据，里面会包含电话记录，短信记录，邮件等等。输入find . -name *.db 命令。  

	iPod-touch:/ root# find . -name *.db
	./Library/Application Support/BTServer/pincode_defaults.db
	./System/Library/Frameworks/CoreLocation.framework/Support/factory.db
	./System/Library/Frameworks/CoreLocation.framework/Support/timezone.db
	./System/Library/Frameworks/CoreTelephony.framework/Support/lasdcdma.db
	./System/Library/Frameworks/CoreTelephony.framework/Support/lasdgsm.db
	./System/Library/Frameworks/CoreTelephony.framework/Support/lasdlte.db
	./System/Library/Frameworks/CoreTelephony.framework/Support/lasdscdma.db
	./System/Library/Frameworks/CoreTelephony.framework/Support/lasdumts.db
	./System/Library/PrivateFrameworks/AppSupport.framework/CityInfo.db
	./System/Library/PrivateFrameworks/AppSupport.framework/Dutch.lproj/Localizable_Places.db
	./System/Library/PrivateFrameworks/AppSupport.framework/English.lproj/Localizable_Places.db
	./System/Library/PrivateFrameworks/AppSupport.framework/French.lproj/Localizable_Places.db
	./System/Library/PrivateFrameworks/AppSupport.framework/German.lproj/	Localizable_Places.db
	./System/Library/PrivateFrameworks/AppSupport.framework/Italian.lproj/	Localizable_Places.db
	./System/Library/PrivateFrameworks/AppSupport.framework/Japanese.lproj/Localizable_Places.db
	./System/Library/PrivateFrameworks/AppSupport.framework/Spanish.lproj/Localizable_Places.db
	./System/Library/PrivateFrameworks/AppSupport.framework/ar.lproj/Localizable_Places.db
	./System/Library/PrivateFrameworks/AppSupport.framework/ca.lproj/Localizable_Places.db
	./System/Library/PrivateFrameworks/AppSupport.framework/calldata.db
	./System/Library/PrivateFrameworks/AppSupport.framework/cs.lproj/Localizable_Places.db
	./System/Library/PrivateFrameworks/AppSupport.framework/da.lproj/Localizable_Places.db
	./System/Library/PrivateFrameworks/AppSupport.framework/el.lproj/Localizable_Places.db
	./System/Library/PrivateFrameworks/AppSupport.framework/en_AU.lproj/Localizable_Places.db
	./System/Library/PrivateFrameworks/AppSupport.framework/en_GB.lproj/Localizable_Places.db
	./System/Library/PrivateFrameworks/AppSupport.framework/es_MX.lproj/Localizable_Places.db
	./System/Library/PrivateFrameworks/AppSupport.framework/fi.lproj/Localizable_Places.db
	./System/Library/PrivateFrameworks/AppSupport.framework/fr_CA.lproj/Localizable_Places.db
	./System/Library/PrivateFrameworks/AppSupport.framework/he.lproj/Localizable_Places.db
	./System/Library/PrivateFrameworks/AppSupport.framework/ru.lproj/Localizable_Places.db
	./System/Library/PrivateFrameworks/AppSupport.framework/sk.lproj/Localizable_Places.db
	./System/Library/PrivateFrameworks/AppSupport.framework/sv.lproj/Localizable_Places.db
	./System/Library/PrivateFrameworks/AppSupport.framework/th.lproj/Localizable_Places.db
	./System/Library/PrivateFrameworks/AppSupport.framework/tr.lproj/Localizable_Places.db
	./System/Library/PrivateFrameworks/AppSupport.framework/uk.lproj/Localizable_Places.db
	./System/Library/PrivateFrameworks/AppSupport.framework/vi.lproj/Localizable_Places.db
	./System/Library/PrivateFrameworks/AppSupport.framework/zh_CN.lproj/Localizable_Places.db
	./private/var/Keychains/keychain-2.db
	./private/var/mobile/Containers/Bundle/Application/A3EBCEF7-D99C-4417-9813-E6CDF5B8DA75/MojiWeather.app/location.db
	./private/var/mobile/Containers/Bundle/Application/A3EBCEF7-D99C-4417-9813-E6CDF5B8DA75/MojiWeather.app/mojicity.db

随便看一个数据库，先下个pp助手，再去Cydia里安装Apple File Conduit "2". 
成功后重启设备，连接pp助手。  mac上按个sqlite数据库查看软件，我的是MesaSqlite. 完后你就能想看什么看什么啦~~  
####注意：图片只是例子，实际情况有所不同
![](http://highaltitudehacks.com/images/posts/ios10/21.png)  
![](http://highaltitudehacks.com/images/posts/ios10/22.png)  

##Gathering information from plist files.  
![](http://highaltitudehacks.com/images/posts/ios10/p.png)

