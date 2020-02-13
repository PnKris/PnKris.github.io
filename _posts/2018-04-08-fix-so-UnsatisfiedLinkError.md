---
title: Android APP运行时报so库加载失败
categories: Android
tags: [Android]
date: 2018-04-08
---

遇到一个比较奇怪的bug：

	# main(1)
	java.lang.UnsatisfiedLinkError
	Couldn't load opencv_java3 from loader dalvik.system.PathClassLoader[DexPathList[[zip file "/data/app/com.lehu.mystyle.excavator-2.apk", zip file "/data/user/0/com.lehu.mystyle.excavator/code_cache/secondary-dexes/com.lehu.mystyle.excavator-2.apk.classes2.zip"],nativeLibraryDirectories=[/data/app-lib/com.lehu.mystyle.excavator-2, /vendor/lib, /system/lib]]]: findLibrary returned null
	java.lang.Runtime.loadLibrary(Runtime.java:358)
	java.lang.System.loadLibrary(System.java:526)
	com.psoft.cv.cvlib.PSCVLibrary.<clinit>(PSCVLibrary.java:35)


看到这个报错第一反应就是.so库没有添加到工程里面，但是发现libs下面已经添加了这个库。
进一步查看这个问题，想到是不是Gradle配置导致了so库没有编译进apk中导致的，将打包好的apk解压发现如下的目录：

<figure class="center">
	<img src="/images/20180408114215.png" alt="">
	<figcaption>lib下的目录</figcaption>
</figure>

发现里面有两个目录：

	armeabi
	armeabi-v7a

而opencv_java3这个库存在于armeabi，armeabi-v7a中并没有，具体如下：
armeabi目录：

<figure class="center">
	<img src="/images/20180408115435.png" alt="">
	<figcaption>armeabi目录</figcaption>
</figure>

armeabi-v7a目录内容：

<figure class="center">
	<img src="/images/20180408115628.png" alt="">
	<figcaption>armeabi-v7a目录</figcaption>
</figure>

<!--more-->

导致找不到opencv-java3的原因就是：我们在lib目录下存在着armeabi-v7a这个目录，而系统去lib下面去查找opencv-java3.so的时候发现存在了armeabi-v7a这个目录就直接去这个目录下面查找，如果找不到就直接返回null，导致应用在运行的时候找不到这个so。

修改办法：将ijkplayer的库放到armeabi目录下，也就是compile的时候采用v5这个版本：

    api 'tv.danmaku.ijk.media:ijkplayer-java:0.8.8'
    api 'tv.danmaku.ijk.media:ijkplayer-armv5:0.8.8'

再重新编译发现lib目录下就只有armeabi一个目录了，运行也不会报错


