---
title: React Native 开发中所遇到的问题
categories: 前端
tag: ['React native']
date: 2019-10-21
---

入坑`React Native`半年多了，平时也会遇到一些问题，但是解决之后，下次遇到可能就记不得，现在已经想不起之前遇到的一些问题了，所以想记录下来，方便以后可以回顾，如果有遇到类似问题的人，也可以有一个参考，这篇博客会不定期累加问题。


## 1.Task :app:compileDebugJavaWithJavac FAILED

 一般导致这种错误的情况会有很多，大部分时候，我在发生这种错误时，在控制台启动项目的时候会报出错误的文件，或者执行`gradlew compileDebugJavaWithJavac ` 命令也可以看到具体的错误信息,利用`Android Studio`查看错误的文件，可以看到具体是哪行发生了错误。

 一般我遇到的错误，大部分是引用错误，在升级了`AndroidX`之后，可是之前引用的一些包会不兼容，需要把android引用改为androidx的。

 这些错误一般都是由于引用的第三方库里发生的错误，也就是`node_modules`中的文件，每次npm之后，node_modules中的文件就会更新，所以每次去改就很麻烦。可以把第三方库fork到自己的git仓库中，更改代码，`package.json`中的地址就引用自己Git仓库中的地址。
 >使用一些第三方库的时候，有时因为第三方库并不是一直在维护的，有可能集成进去，会有 `sdk` 兼容问题，所以要把sdk版本改为和自己项目中的一致，也可以fork到自己的仓库中，避免每次修改。



## 2.[react-native-camera](https://github.com/react-native-community/react-native-camera) 集成问题实现扫码

安装`react-native-camera`之后，会出现错误，我通过[这个](https://github.com/react-native-community/react-native-camera/issues/2150)找到了解决办法。(PS: 最好从官方文档看教程，我之前就是在网上搜的教程，但是已经过时了，这样就浪费时间找问题出在哪，网上的教程作为一个参考点就行)
但是最后这个库实现的扫码，速度没有达到我们的要求(正常手机扫码速度还可以，但是我们要在性能很差的平板上快速扫码就有点难度了)，所以放弃了这种方式，最后使用**PDA扫码**和集成**红外摄像头扫码**(通过使用java代码,以原生的方式实现，辛亏有大佬帮忙ヽ(○^㉨^)ﾉ♪)


## 3.[react-native-ble-manager](https://github.com/innoveit/react-native-ble-manager#methods) 连接蓝牙

之前用的测试机是 `Android5.1` 的，能够正常扫描蓝牙，一直以为就这样就算正确使用这个插件连接蓝牙，但是之后用自己的手机运行app的时候，发现怎么都扫描不到蓝牙，从网上搜索得知，扫码蓝牙还需要开启位置权限
```js
<uses-permission android:name="android.permission.BLUETOOTH" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
<uses-feature android:name="android.hardware.bluetooth_le" android:required="true"/>
```
需要加入这些权限，但是我查看自己的`AndroidManifest.xml`文件时，发现这些权限已经加上去了。原来，在`Android6`之后，权限不仅要在清单文件AndroidManifest.xml里面申请,还有单独调用api，去让用户选择是否同意你申请这个权限。
```js
 _checkLocation =()=>{
        if(Platform.OS === 'ios'){
            return false;
        }
        const granted = PermissionsAndroid.check(PermissionsAndroid.PERMISSIONS.ACCESS_COARSE_LOCATION)

        granted.then((data) => {
            console.log('data----', data)
            if (!data) {
                this.requestLocationPermission()
            }
        }).catch((err) => {
            console.log('err---------', err.toString())
        })
    }
    //申请地址权限
    async requestLocationPermission() {
        try {
            const granted = await PermissionsAndroid.request(
                PermissionsAndroid.PERMISSIONS.ACCESS_COARSE_LOCATION,
                {
                    //第一次请求拒绝后提示用户你为什么要这个权限
                    'title': '是否允许地址查询权限',
                    'message': '此权限会造成系统异常，请允许',
                    buttonNeutral: '等会再问我',
                    buttonNegative: '不行',
                    buttonPositive: '好吧',
                }
            )

            if (granted === PermissionsAndroid.RESULTS.GRANTED) {
                console.log("你已获取了定位权限")
            } else {
                console.log("获取定位权限失败,会造成系统异常")
            }
        } catch (err) {
            console.log(err.toString())
        }
    }
```
所以可以通过以上代码，获取位置权限。

## 4.native-echarts 经常加载不出
因为对`echarts`很熟悉，所以要在react-native绘制图表时，就选择了使用echarts封装的[native-echarts](https://github.com/somonus/react-native-echarts#readme)，但是这个库已经很久不维护了，所以就fork到自己的git地址中，修改了一些地方引用。

但是使用这个图表，有时候会加载不出来，我在网上搜索也并没有搜到解决办法，网上只有说在Android中显示不出来,把`tpl.html`文件复制到`android/app/src/main/assets`文件夹下。我也在交流群中问过一些人，他们好像都没有遇到类似问题，这让我很费解，也许是因为我需要实时刷新的缘故，也就没有找到解决办法了。

native-echarts中使用echarts的办法是通过`Webview`中引用html文件实现的，我给webview加了`startInLoadingState`属性，可以查看加载状态，在我实时刷新的时候，刷新次数过多，状态会一直处于加载中，就加载不出来了，不清楚是不是由于性能消耗过多，暂时没有解决。

所以就换了一个图表库，[react-native-charts-wrapper](https://github.com/wuxudong/react-native-charts-wrapper),这种图表库是基于原生的，所以性能提高了很多，图表能立马加载出来，但也有一定局限性，只有八种图表，配置项也没有echarts那么多。

要使用这个图表，需要配置一些东西，在android中配置项较少，就不说了，`ios`中的配置可以参考[这里](https://www.jianshu.com/p/432517c5b531),但其中一些知识点已经过时不需要了，所以正确步骤还是需要参照[官方文档](https://github.com/wuxudong/react-native-charts-wrapper/blob/master/installation_guide/README.md),但是官方文档没有配图，所以不懂的地方可以参考上面这个参考，有图片引导，更加易懂一点。

(PS： `Charts`应该使用`3.3.0`版本，但是git官方地址已经更新超过这个版本了，所以下载的时候一定要注意版本一致，我就是因为下载了最新的版本，所以一致对不上。这时候就体现出仔细查看官方文档的好处了ヽ(○^㉨^)ﾉ♪)

## 5.ios出现 Build input file cannot be found: '/Users/mac/Library/Developer/Xcode/DerivedData/jty-ceemylpddhxuyyegcppypjhihevw/Build/Products/Debug-iphonesimulator/jty.app/PlugIns/jtyTests.xctest/jtyTests'

![ios问题](https://www.eyes487.top/fs/uploads/1573386594646-ios1.jpg "图1")

解决方法：
Xcode > File > Workspace Settings...
or
Xcode > File > Project Settings...


Shared Project Settings和Per-User Project Settings 中的Build System都从New Build System (默认) 改为Legacy nuild System

## 6. react-native 从0.57.8升级到0.59.10，在Android中运行，出现  Could not get unknown property 'mergeResourcesProvider' for object of type com.android.build.gradle.internal.api.ApplicationVariantImpl.

![Android问题](https://www.eyes487.top/fs/uploads/1573386561035-Android1.png "图2")

解决方法：
/android/build.gradle  改为classpath 'com.android.tools.build:gradle:3.3.0'
/android/gradle/wrapper/grale-wrapper.properties 改为 distributionUrl=https\://services.gradle.org/distributions/gradle-4.10.1-all.zip

运行有可能会出错，记得清理一下缓存 cd android && gradlew clean

## 7 Android9.0 http无法访问网络问题
先说前提背景，首次使用react-native 0.60.5版本建立项目，在手机上调试的时候没有问题，打包安装在手机上出现闪退。
在Android studio中Logcat上看到闪退报错信息：
### 7.1  .com.facebook.react.common.JavascriptException: console.assert is not a function. (In 'console.assert(null!=o,"'this' is expected an Event object, but got",n)', 'console.assert' is undefined), stack:
    o@112:173
    w@112:1841
    dispatchEvent@112:5597
    value@111:6095
    value@111:2835
    value@44:1280
    value@23:3518
    <unknown>@23:822
    value@23:2772
    value@23:794
    value@-1
![Stack问题](https://www.eyes487.top/fs/uploads/1575181213971-stack.png "图3")
解决办法：参考的 [这里](https://github.com/facebook/react-native/issues/26007)

添加如下信息在 index.js中
```js
if (!__DEV__) {
    global.console = {
        info: () => {},
        log: () => {},
        assert: () => {},
        warn: () => {},
        debug: () => {},
        error: () => {},
        time: () => {},
        timeEnd: () => {},
    };
}
```

通过这些操作，app不闪退了，但是页面还是没有如期显示，是因为并没有访问到网络，没有拿到后台的数据，

### 7.2  在网上搜索了一下，发现是`Android 9`会出现这个问题，我在Android 8测试了一下，发现是的，不会出现这个问题。

解决方法：
* APP改用 `https` 请求
* `targetSdkVersion` 降到27以下
* 在 `AndroidManifest.xml` 文件的 `application` 加入：`android:usesCleartextTraffic="true"`

我采用的是第三种方法

## 8. Attempt to invoke virtual method 'android.graphics.drawable.Drawable android.graphics.drawable.Drawable$ConstantState.newDrawable(android.content.res.Resources)' on a null object reference

使用场景：`react-native 0.57.8`  `Android 9`
我在页面中，使用`TextInput`放入表格中的每个单元格，就出现了这个问题，其他时候是没有出现这个问题的

![drawable问题](https://www.eyes487.top/fs/uploads/1576468673838-drawable.png "图4")

解决方法： 在 `android/src/main/res/values/styles.xml` 插入这句
```js
<item name="android:editTextBackground">@android:color/transparent</item>
```
[参考地址](https://github.com/facebook/react-native/issues/17530)


## 9. Date 问题

在debug模式下，没有问题
如果不开启debugger-ui， Date.parse就获取不到数据，暂不知道原因，用的moment来处理时间

## 10. android.support.v4.content.FileProvider找不到问题

我在使用 `react-native-doc-viewer`的时候遇到这个问题，因为我给项目升级了`androidX`,所以导致和这个库里面不兼容，(ps:同时这个库的sdk版本太旧，需要自己修改匹配自己的项目的才能打包)。
![fileprovider问题](https://www.eyes487.top/fs/uploads/08aa3bccf9f2cde1d7892884db3871ee.png)

解决方法： 找到上图报错的文件夹，修改`android.support.v4.content.FileProvider` 为 `androidx.core.content.FileProvider`
可以把这个库fork到自己的仓库，之后就引用自己仓库的，就不用重装依赖之后，每次都修改了

## 11. 点击Home键退出，再次点击桌面图标启动应用，会再次启动APP，而不是把后台程序显示出来

解决方法一：可以在`AndroidManifest.xml `中修改 `MainActivity` 的 `launchMode`为`SingleTask`
```js
<activity
  android:name="***.MainActivity"
  android:label="@string/app_name"
  android:configChanges="keyboard|keyboardHidden|orientation|screenSize"
  android:windowSoftInputMode="adjustResize"
  android:screenOrientation="portrait"
  android:launchMode="singleTask">
```
解决方式二： 由于 RN 的Activity 继承自 ReactActivity ,所以需要修改源码，在`MainActivity.java`文件中`ReactActivity` 的 `onCreate` 的方法添加,没有onCreate方法，就自己添加一个
```js
// import android.content.Intent; //Intent
//...
protected void onCreate(Bundle savedInstanceState) {
  super.onCreate(savedInstanceState);
    if (!this.isTaskRoot()) {
        Intent mainIntent = getIntent();
        String action = mainIntent.getAction();
        if (mainIntent.hasCategory(Intent.CATEGORY_LAUNCHER) && action.equals(Intent.ACTION_MAIN)) {
            finish();
            return;
        }
    }
  }
//   mDelegate.onCreate(savedInstanceState); //假如你之前有这个方法的话，没有就可以不写
  //PS：记得在 super.onCreate() 方法之后，mDelegate.onCreate之前执行有效
 }
```

## 12. Style property 'marginLeft' is not supported by native animated module

在写轮播`(swiper)`的时候,使用到了`Animated`，最开始是使用的`transform`，没有什么问题
```js
 <Animated.View
    style={[
    styles.content,
    {
        transform: [{
            translateX:  this.state.positionAnimated,
        }],
        // marginLeft: this.state.positionAnimated
    }
    ]}
>
```
但是，transform有一个问题就是，我需要在轮播每一块元素中添加输入框，在第一块区域中是可以操作的，切换到第二块区域，输入框就选不中了，不能定位到具体的元素上，就想着换一种方式，不用平移，用`marginleft`的方式来实现，换成marginleft之后就出现了这个问题

折腾了好久，发现是在 `Animated.timing`中开启了原生动画`useNativeDriver: true`才导致了这个问题，原生动画并不能支持所有的属性，像`transform` 和 `opacity`可以工作，`flexbox`和` position` 就不行

## 13. react-native如何查看网络请求

之前查看网络请求的数据，都是通过在控制台`console.log`的方式，原来react-native也可以像浏览器一样，控制面板`Network`中查看网络请求。

需要使用到这个 [react-native-debugger](https://github.com/jhen0409/react-native-debugger)
* 首先[下载软件包](https://github.com/jhen0409/react-native-debugger/releases)，可以下载zip压缩包，解压之后里面会有 `react-native-debugger.exe`
   **注意：React Native version <= 0.61, 就要使用 v0.10**
* 在使用之前，要确保关闭了 `Remote JS Debugging`,关闭之后，双击`react-native-debugger.exe`打开，在开启`Remote JS Debugging`,就连上这个软件了
* 右键点击软件，会有一个选项`Enable Network Inspect`,这就打开了网络检查，就可以在Network面板中看到发出的网络请求了

!['react-native-debugger'](https://www.eyes487.top/fs/uploads/93e7b02f1d6785ed56b0619cd5bd1355.png)

`react-native-debugger`当然还有很多其他功能，就自己去发掘了

## 14. react-native-vector-icons乱码不显示问题，显示的是方框中一个X

react-native 0.6X版本，都可以自动link，不用手动link，但是安装了react-native-vector-icons之后，图标乱码不显示

解决方法：
* 把 `node_modules\react-native-vector-icons\Fonts`文件夹复制到 `android\app\src\main\assets\fonts` ,把文件夹名称大写要改为小写的
* 然后在 `android/app/build.gradle` 中添加
```js
project.ext.vectoricons = [
    iconFontNames: [ 'Feather.ttf','FontAwesome.ttf','AntDesign.ttf' ] // 指定要使用的字体库。
]
apply from: "../../node_modules/react-native-vector-icons/fonts.gradle"
```

修改了原生文件，重启项目，就可以显示出来了

## 15. Execution failed for task ':app:mergeDexDebug'

一般出现这种操作，是由于`Multidex builder`,在yaml文件找那个导入了很多无法放入单个.dex文件的软件包时，通常就会出现这种问题，因此需要启用 `multidex`

说一下我的情况，`RN 0.63`在使用`react-native-pdf`的时候出现这个问题

解决方法：在 `android/app/build.gradle`加入
```js
android {
    defaultConfig {
        //...
        multiDexEnabled true
    }
}

```

## 16. 有时候会出现，在真机调试的时候没有问题，但是打包release之后，app会出现闪退的问题

如果app发生了错误，出现闪退，， 可以在项目目录下通过控制台使用 `adb logcat AndroidRuntime:E *:S`命令，或者在android Studio里面可以看到app运行时的错误

## 17. java.lang.UnsatisfiedLinkError: Can't obtain peer field ID for class com.sun.jna.Pointer

在真机调试的情况下，没有出现问题，我打包成release版本就出现散退，通过`adb logcat AndroidRuntime:E *:S`命令看到了运行时的错误，就是上面这个信息。
出现这个问题的原因是，在打包时开启了 `def enableProguardInReleaseBuilds = true`，但是一些第三方包是不需要混淆的，所以可以配置忽略混淆第三方jar包

在`android/app/proguard-rules.pro`文件中写上
```bash
//...

# Add any project specific keep options here:

-keep class com.sun.jna.** { *; }
-keep class com.caysn.autoreplyprint.** { *; }
```