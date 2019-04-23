## 优化目标
只要app在运行，肯定是会消耗电量的。所以在优化的时候，应该重点关注异常的耗电行为。
经过资料查询
异常耗电一般分为
	1、CPU异常（例如不正确的wakelock使用）
	2、网络异常（频繁的网路请求）。
	3、传感器异常（不需要使用GPS的功能，却监听了GPS）


## 如何量化

### 如何得出耗电量
一个应用运行时，操作系统也在运行，其他后台程序也在运行。怎么去计算我们当前应用的耗电量呢？

 Android 在4.1版本后在系统增加了battry info模块，记录一定时间周期内整机的功耗状态以及每个应用的功耗详情。
### 如何衡量应用耗电
怎么去衡量一个应用耗电多少？肯定是运行应用一段时间，然后计算电量消耗。但是，一个应用通常都有许多模块，如果要全局的计算一个应用耗电，会很麻烦，也不方便测试。所以，我们应该以应用的模块为单位，在一定时间内反复使用该模块，然后得出电量消耗。比如，要测试视频播放的电量消耗。我们可以播放视频一小时，然后通过battry info查看电量消耗。  **所以我们最终的衡量标准，应该是以模块为标准，分别衡量**。
![耗电记录表.png](https://upload-images.jianshu.io/upload_images/9243886-eb3ddf6deee8ae50.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 工具使用介绍
获取系统耗电量。android4.1后增加了电池信息，所以我们只要dump下来就可以了。
```adb shell dumpsys batterystats```
文件是这样。这个看起来真是头大。
![image.png](https://upload-images.jianshu.io/upload_images/9243886-929731d730826c46.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

幸好，google开源了一款电量分析工具Battery Historian，可视化电池信息。 用了之后我们看到的是这样。

![image.png](https://github.com/ZhangHao555/ReadingNotes/blob/master/pics/%E7%94%B5%E9%87%8F%E6%B6%88%E8%80%97%E5%9B%BE.png)

#### 使用Battery Historian
##### 导出数据电池
```
adb shell dumpsys batterystats --reset   清除设备电量使用记录
adb shell dumpsys batterystats --enable full-wake-history  获得应用程序的wakelock记录
进行相应的测试
adb shell dumpsys batterystats --disable full-wake-history 关闭应用程序的wakelock记录
adb bugreport bugreport.zip 导出记录
自己搭建Battery Historian 服务 或者使用 https://bathist.ef.lc/ 上传记录。
```

## 举例，对视频播放模块进行耗电优化。
1、清除电量使用记录 adb shell dumpsys batterystats --reset
2、获得应用程序的wakelock记录 adb shell dumpsys batterystats --enable full-wake-history
3、播放三十分钟视频
4、adb shell dumpsys batterystats --disable full-wake-history  关闭应用程序的wakelock记录
5、adb bugreport bugreport.zip 导出
6、上传 https://bathist.ef.lc/ 进行分析

![](https://github.com/ZhangHao555/ReadingNotes/blob/master/pics/historian1.png)
![](https://github.com/ZhangHao555/ReadingNotes/blob/master/pics/historian21.png)
### 分析数据
第一张图 以图表的形式给用户一种直观的感受。可以看出电量在什么阶段下降最快，进而分析出异常的情况。

第二张图，左边可以选择app来查看app的使用情况。可以发现，在这一个小时里，
a、我们播放视频消耗了17.60%的电量。
b、cpu一直处于唤醒状态
c、wifi在22分钟内传输了236.68MB流量

## GOOGLE的优化建议
使用工具可以帮助我们快速定位异常的耗电情况。但是却不容易发现一些不太明显的多余耗电操作（网络多请求了一次，数据没有压缩等等）。所以，我们还应该借鉴经验，根据实际情况进行优化。

对于优化电量，Google给了我们三个建议

> There are three important things to keep in mind in keeping your app power-thrifty:
>1、Make your apps Lazy First.
2、Take advantage of platform features that can help manage your app's battery consumption.
3、Use tools that can help you identify battery-draining culprits.

我们可以根据这些建议来进行优化。
### 1、Lazy First
减少: 减少多余的操作，例如可以通过缓存来减少减少网络访问和计算。通过合理的设计来减少接口的请求。例如可以通过压缩来减少数据的传输。

推迟: 如果任务不是那么紧急，我们可以推迟任务，等待设备在充电的时候再执行。例如上传用户数据，后台更新等任务。

合并:多个任务是否可以合并到一起执行，避免多次唤醒cpu和网络。

对应到我们的项目中值得优化的点：
#### a、减少： 

经过产品环境抓包发现，我们的请求似乎返回是没有经过压缩的。 
GZIP现今已经成为Internet 上使用非常普遍的一种数据压缩格式。一般对纯文本内容可压缩到原大小的40％
![image.png](![](https://github.com/ZhangHao555/ReadingNotes/blob/master/pics/search_api.png))

目前App内和server下发的图片基本都是png格式。但是对于JPEG、PNG、GIF等常用图片格式的优化已几乎达到极致，因此Google于2010年提出了一种新的图片压缩格式 — WebP，给图片的优化提供了新的可能。
WebP为网络图片提供了无损和有损压缩能力，同时在有损条件下支持透明通道。据官方实验显示：无损WebP相比PNG减少26%大小；有损WebP在相同的SSIM（Structural Similarity Index，结构相似性）下相比JPEG减少25%~34%的大小；有损WebP也支持透明通道，大小通常约为对应PNG的1/3。

项目中的消息通知是采用轮询实现的，每隔一分钟去请求服务器获取最新消息。 我们可以集成第三方推送工具，来实现消息推送，避免频繁唤醒网络。

#### b、推迟
针对不太紧急的任务，我们可以推迟任务等到相应的情景再触发。例如，应用的更新。我们在用户进行充电，并且连接wifi的时候自动进行更新。 
Android 从5.0开始 提供了JobScheduler，我们可以使用它在特定情况下例如充电状态，wifi连接状态执行任务。
这是JobScheduler支持的触发条件。

![image.png](https://upload-images.jianshu.io/upload_images/9243886-637b9d946f8f7e07.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



#### c、合并
Can work be batched, instead of putting the device into an active state many times? For example, is it really necessary for several dozen apps to each turn on the radio at separate times to send their messages? Can the messages instead be transmitted during a single awakening of the radio?
这个主要是针对后台app的，操作系统已经帮我们做了优化。

### 2、Take advantage of platform features that can help manage your app's battery consumption.
Android 5.0 为我们提供了 JobScheduler ，用于优化系统运行，提供资源利用率，节省电量。
Android 6.0 新增Doze模式和App Standby模式   ： 
Doze模式通过在设备长时间处于闲置状态时推迟应用的后台CPU和网络Activity来减少电池消耗
App Standby 应用切换到后台，并长时间没有与用户交互时，系统便会进入App Standby模式，限制CPU和网络来减少电池消耗。
Android 9.0 新增App Standby Buckets  系统根据用户使用行为使用机器学习算法来将应用放置在不同的Buckets里面。系统根据buckets的类型来限制应用行为。

![image.png](https://upload-images.jianshu.io/upload_images/9243886-4d762b27623a15c6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**这里需要注意** ，因为平台的新特性，可能会导致应用的功能出现异常。比如我们应用中的消息轮询，当应用进入App Standby  模式后，网络请求会被推迟到，导致某个时间点发出大量的请求。![image.png](https://upload-images.jianshu.io/upload_images/9243886-d1494156f444aa81.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/9243886-fc94f7e42049df67.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




### 3、Use tools that can help you identify battery-draining culprits.
使用工具分析 ?[Battery Historian](https://github.com/google/battery-historian)

## 总结
电量优化其实是一个很大的概念。拆分开来，就是内存优化、CPU优化、网络优化、UI优化、传感器优化。

总体来说，对于电量优化我们有两个事情要做。
1、分模块量化，排查异常耗电，解决异常耗电。
2、写代码时要有意识的去测试UI、内存、CPU、网络等参数，养成良好的习惯






















