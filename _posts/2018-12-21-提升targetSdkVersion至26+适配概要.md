---
layout:     post
title:      提升targetSdkVersion至26+适配概要
subtitle:   
date:       2018-12-21
author:     lx8421bcd
header-img: img/post-bg-android.jpg
catalog: true
tags:
    - Android
    - Android开发随笔
---

### 前言
前几天发版时接到了华为那边的提醒，说请尽快将targetSdkVersion提升到26+，2019年5月1号之后将会拒绝所有targetSdkVersion低于26的应用的上架和更新。于是查了一下，发现目前国内的主要应用渠道商都已经签订了[电信终端产业协会（TAF）发布《移动应用软件高API等级预置与分发自律公约》](http://www.taf.net.cn/News_detail.aspx?_NOTICE_ID=231)。  

这个事情似乎没在国内掀起什么舆论，然而这确实是对国内Android用户的重大利好消息，利用android向下兼容，不适配新版本特性进行一些流氓行为的日子即将一去不复返了。  
提升targetSdkVersion到26+意味着，必须进行一系列权限适配和兼容，这里也就记录一下我所遇见的兼容问题以及处理方案。  


### 运行时权限
如果你的应用之前的targetSdkVersion < 23，那么升级targetSdkVersion到26+首先要做的就是适配运行时权限。  
Android 6.0引入了运行时权限机制，这已经过去2年多了，适配相关的文章、库已经有很多，这里就不再赘述。仅记录一下一些适配经验。 

__SD卡读写__ 权限是是运行时权限中最为零碎和麻烦的权限管理，以至于很多应用直接启动时申请如果拒绝就退出。这样一刀切虽然可以保证改动最小化且应用正常运行，但体验很明显是欠妥的。  
其实应用中的很多文件操作内容可以放到系统SDK提供的内部和外部路径下   
```java
/** 缓存路径，用于临时存放，可以随时清除 **/
// 内部缓存路径
context.getCacheDir();
// 外部缓存路径，使用前最好检查SD卡是否挂载
context.getExternalCacheDir();

/** 文件路径，用于存放长期有效的文件 **/
// 内部文件路径
context.getFilesDir();
// 外部文件路径
context.getExternalFilesDir()
```
编辑这两个路径下的文件是不需要获取SD卡读写权限的，因此我们压缩图片、生成中间文件、存储log日志等操作生成的文件都可以放在这个路径下。__需要特别注意的是下载安装的apk最好放在外部存储中__，比如应用内更新下载的apk，因为内部缓存路径默认只有本应用才有读写权限，除非调用设置开启，如果将apk放在内部存储中又不做特殊处理，会导致系统的PackageInstaller无法读取apk进而安装失败。

另外需要注意的是并不是只有直接的文件操作才会需要申请SD卡读写权限，使用ContentResolver、读取一些媒体库也需要这类权限，在适配时需要仔细检查。


### 浮窗
在Android 6.0已经将浮窗划为运行时权限，而且与上面的运行时权限不大相同，属于特殊权限，浮窗权限需要使用Intent唤起应用设置页面让用户开启，然后在```onActivityResult()```中判断是否拥有权限。```WRITE_SETTINGS```的申请方法与此相同
```java
// 判断是否拥有浮窗权限
boolean hasPermission = Settings.canDrawOverlays(activity);

// 跳转权限申请界面
Intent intent = new Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION);
intent.setData(Uri.parse("package:" + activity.getPackageName()));
activity.startActivityForResult(intent, MANAGE_OVERLAY_PERMISSION_REQUEST_CODE);

```
在Android 8.0中，对于浮窗进行了更严格的区分和限制 [官方文档](https://developer.android.com/about/versions/oreo/android-8.0-changes?hl=zh-cn)

    提醒窗口
    使用 SYSTEM_ALERT_WINDOW 权限的应用无法再使用以下窗口类型来在其他应用和系统窗口上方显示提醒窗口：

    TYPE_PHONE
    TYPE_PRIORITY_PHONE
    TYPE_SYSTEM_ALERT
    TYPE_SYSTEM_OVERLAY
    TYPE_SYSTEM_ERROR
    相反，应用必须使用名为 TYPE_APPLICATION_OVERLAY 的新窗口类型。

    使用 TYPE_APPLICATION_OVERLAY 窗口类型显示应用的提醒窗口时，请记住新窗口类型的以下特性：

    应用的提醒窗口始终显示在状态栏和输入法等关键系统窗口的下面。
    系统可以移动使用 TYPE_APPLICATION_OVERLAY 窗口类型的窗口或调整其大小，以改善屏幕显示效果。
    通过打开通知栏，用户可以访问设置来阻止应用显示使用 TYPE_APPLICATION_OVERLAY 窗口类型显示的提醒窗口。

因此需要增加额外的浮窗判断
```java
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    wmParams.type = WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY;
}
else {
    wmParams.type = WindowManager.LayoutParams.TYPE_PHONE;
}
```


### 应用安装
在Android 8.0以后，Google对第三方app安装apk进行了严格的限制，新增了```android.permission.REQUEST_INSTALL_PACKAGES```权限，如果你的应用想要获得安装apk的权限，必须在manifest中声明此权限，并且在需要调用的时候执行类似于浮窗权限申请一样的权限申请流程。
由于Android向下兼容的关系，targetSdkVersion低于26则不用关心这个，但是现在我们需要适配到26+，如果我们有应用内更新或者下载安装其他apk的需求，就必须要关注安装权限了。
```java
// 权限判断
public static boolean hasInstallPackagePermission() {
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.O) {
        return true;
    }
    return context.getPackageManager().canRequestPackageInstalls();
}

// 权限申请
Intent intent = new Intent(Settings.ACTION_MANAGE_UNKNOWN_APP_SOURCES);
intent.setData(Uri.parse("package:" + activity.getPackageName()));
activity.startActivityForResult(intent, requestCode);
```

### FileUriExposedException
直接将老项目的targetSdkVersion提到24+，在读取媒体库文件、安装apk的时候会出现崩溃，报错```FileUriExposedException```。其实这也是老项目欠下的适配债了，Google认为通过诸如```file://URI```这样的URI访问文件是不安全的，特别是访问其它应用的私有目录和文件，因此很早就提供了```FileProvider```这样的东西用于管理文件访问，只不过由于向下兼容，老方法一直能用，关注寥寥。  
在Android 7.0+的系统上，Android SDK的 StrictMode 不再允许在应用外部公开```file://URI```，如果携带```file://URI```离开自己的应用（访问PackageInstaller，访问相册 etc.），就会抛出```FileUriExposedException```。

对于这个问题有两种解决方案
1. 老老实实的编写FileProvider  
如何设置FileProvider网上已经有很多文章，这里不再赘述  

2. 重新设置StrictMode，让VM忽略URI检查
```java
// 在Application onCreate()期间执行
// 关闭FileUriExposedException检查
StrictMode.VmPolicy.Builder builder = new StrictMode.VmPolicy.Builder();
StrictMode.setVmPolicy(builder.build());
```
此方法为暂缓之计，如果适配工作量极大可以先行使用，建议逐步适配过渡到FileProvider。  


### Notification Channel    
Android 8.0以后增加了推送渠道（Notification Channel）的概念，用于更精细的划分一个应用的不同推送，比如一个类似知乎这样的以阅读为主的综合类应用，就可以将推送分为推广、私信等channel；用户不想看推广推送，又怕错过私信，就可以将推广关闭，保留私信的channel。  
我们知道Android Notification有3个必填属性：icon、title、content，这三个如果缺了任何一个都将会导致Notification不显示，而对于targetSdkVersion=26+的应用，又增加了一个NotificationChannelId，如果构建Notification的时候不设置NotificationChannelId，后果会比不设置前三个还严重，将会在显示notification时崩溃。所以适配26+我们必须对应用的NotificationChannel进行管理。

Notification Channel由开发者创建，构建时的配置与普通Notification相似，可以设定重要等级、是否震动、是否响铃等，
```java
public static NotificationCompat.Builder buildNotification(String channelId, String channelTitle) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            if (TextUtils.isEmpty(channelId) || TextUtils.isEmpty(channelTitle)) {
                channelId = DEFAULT_CHANNEL_ID;
                channelTitle = DEFAULT_CHANNEL_NAME;
            }
            NotificationManager mNotificationManager = getNotificationManager();
            NotificationChannel mChannel = mNotificationManager.getNotificationChannel(channelId);
            if (mChannel == null) {
                mChannel = new NotificationChannel(channelId, channelTitle, NotificationManager.IMPORTANCE_DEFAULT);
                mChannel.setSound(null, null);
                mNotificationManager.createNotificationChannel(mChannel);
            }
            return new NotificationCompat.Builder(mContext, mChannel.getId());
        }
        return new NotificationCompat.Builder(mContext);
    }
```
__需要注意的是NotificationChannel一旦创建，则不再能够被应用修改，只有用户可以修改__。所以在调试NotificationChannel时，记得卸载重装。


### 后台Service
Android 8.0对应用启动后台服务进行了严格的限制，对于targetSdkVersion=26+的应用，系统不允许后台应用创建后台服务，必须以通知栏可见的形式告诉用户你正在后台执行任务。也就是使用```ContextCompat.startForegroundService()```创建前台服务，然后必须在5秒内调用此服务的startForeground()方法，否则系统将立即停止服务，抛出```RemoteServiceException```异常  

    android.app.RemoteServiceException: Context.startForegroundService() did not then call Service.startForeground()
    ......

在Android 9.0之后，应用想要启动前台服务，必须要拥有```android.permission.FOREGROUND_SERVICE```权限，否则会抛出异常，这个权限等级并不高，在Manifest中声明即可；如果你的targetSdkVersion=28+，又需要使用前台进程，请记得声明此权限。



### 非SDK接口访问限制
在Android P中Google正式对使用反射调用Android SDK内部不对外开放的接口来达成一些目的的行为作出限制。 (对非 SDK 接口的限制 
 | android developer)[https://developer.android.com/about/versions/pie/restrictions-non-sdk-interfaces?hl=zh-cn]  
这意味着以前很多通过反射实现的小花样即将告一段落，一般我们的开发中很少有这样的需求，但老项目中或多或少会有历史遗留代码，因此也需要注意。  
在适配这一块时，需要注意到Google所列出的几个“SDK名单”  

    用于限制非 SDK 接口的不同名单有哪些，它们在限制性行为方面的对应含义是什么？
    下面是名单类型：
    白名单：SDK
    浅灰名单：仍可以访问的非 SDK 函数/字段。
    深灰名单：
        对于目标 SDK 低于 API 级别 28 的应用，允许使用深灰名单接口。
        对于目标 SDK 为 API 28 或更高级别的应用：行为与黑名单相同
    黑名单：受限，无论目标 SDK 如何。 平台将表现为似乎接口并不存在。 例如，无论应用何时尝试使用接口，平台都会引发 NoSuchMethodError/NoSuchFieldException，即使应用想要了解某个特殊类别的字段/函数名单，平台也不会包含接口。
    ...

如果你的应用涉及到浅灰名单，可以暂时不用担心，主要的适配工作在深灰名单和黑名单上，至于怎么适配，其实也没啥好说的，无非就是有替代方案的替代，替代不了的改需求呗……


### 第三方SDK
其实……提升targetSdkVersion最头疼的，不是自己的代码，毕竟自己的代码怎么改都行，而是依赖的第三方library，特别是那些早已不再维护的第三方SDK，头都疼掉……  
这一块能做的，只有3点  
1. 升级到最新版SDK
2. 在SDK提供的接口中寻找有没有兼容方法
3. 怼SDK提供方，让他们做新版的适配，如果能怼出来，也算是功德一件了😄