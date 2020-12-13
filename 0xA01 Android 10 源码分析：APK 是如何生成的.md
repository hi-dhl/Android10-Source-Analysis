# 0xA01 Android 10 源码分析：APK 是如何生成的

![android-APK-banner-670x335-1](http://cdn.51git.cn/2020-04-25-android-apk-banner-670x335-1.jpg)

## 前言

> * 这是 Android 10 源码分析系列的第 1 篇
> * 分支：android-10.0.0_r14
> * 全文阅读大概 5 分钟

APK 的文件可以分为 **代码** 和 **资源** 两部分，接下来源码分析系列，会完全围绕着，这两部分内容来分析，而今天这篇文章是 Android 10 源码分析系列的第 1 篇。

本文预计会分为两篇文章来分析 APK 是如何生成的：

* 从原理的角度分析 APK 是如何生成的
* 如果不使用 AndroidStudio 如何生成 APK

我们很多时候都是直接点击 Android Studio 中直接点击 `Run ‘app’`，就可以在 `build/outputs/apk` 目录下生成 APK 文件，那么 Android Studio 是如何做到的呢？

接下来我们一起来分析一下 APK 的构建过程，APK 的文件可以分为 **代码** 和 **资源** 两部分，那么构建 APK 的过程中，也会对 **代码** 和 **资源** 做分别的处理。

我们先来看看 **Google提供的流程图** 大概了解一下 APK 的构建过程


**新版构建流程图**

![15813472877044-w350](http://cdn.51git.cn/2020-02-12-15813472877044.png)


APK 打包的内容主要有：

* 应用模块用到的源代码、资源文件、aidl 接口文件等等
* 依赖模块即源代码即第三方依赖库如：aar、jar、so 文件等等

新版构建流程图只是描述了大概的过程，为了能够清楚的了解 APK 是如何生成的, 在来看一下老版构建流程图。

**老版构建流程图**
![2019-03-22-15532697195669-w350](http://cdn.51git.cn/2020-02-12-2019-03-22-15532697195669.png)


我们先来了解一下图中所示各个工具的作用。

| 名字 | 功能 |
| --- | --- |
| AAPT/APT2| Android 资源打包工具 | 用于将资源文件编译成二进制文件，<br/> 也可以用来查看 APK 的一些信息 |
| AIDL | 将所有的 AIDL 接口转化为 Java 接口  |
| Javac(Java Compiler) | 将所有的 Java 代码编译成 Class文件  |
| Dex | 将 Class 文件编译成 Dex 文件 |
| Apkbuilder | 将处理后的资源和代码打包生成 APK 文件 |
| Jarsigner/Apksigner | 对未签名的 APK 文件进行签名 |
| Zipalign | 优化签名后的 APK，减少运行时所占用的内存 |

## 构建过程

Apk 的构建过程大概分为如下几步：

1. 使用 AAPT 工具生成 R.java 文件
2. 所有的 AIDL 接口转化为 Java 接口
3. 将 Java 代码编译成 Class 文件
4. 将 Class 文件编译成 Dex 文件
5. 打包生成 APK 文件
6. 对 APK 文件签名
7. 优化 APK 文件

#### 1. 使用 AAPT 工具生成 R.java 文件

AAPT（Android Asset Packaging Tool）android 资源打包工具，将资源文件（包括AndroidManifest.xml、布局文件、各种 xml 资源等）打包生成 R.java 文件，将 AndroidManifest.xml 生成二进制的 AndroidManifest.java 文件<br/>

```
aapt p -M AndroidManifest.xml -S output/res/ -I android.jar -J ./ -F input/out.apk

p：打包
-M：AndroidManifest.xml 文件路径
-S：res 目录路径
-A：assets 目录路径
-I：android.jar 路径，会用到的一些系统库
-J 指定生成的 R.java 的输出目录
-F 具体指定 APK 文件的输出
```

但是从 Android Studio 3.0 开始，google 默认开启了 AAPT2 作为资源编译的编译器，AAPT2 的出现为资源的增量编译提供了支持，aapt2 主要分两步，compile 和 link <br/>

**compile**

```
aapt2  compile -o res.apk --dir output/res/

-o：指定已编译资源的输出路径
--dir：指定包含多个资源文件的资源目录
```

**link**

```
aapt2 link -o input/out.apk -I tools/android.jar --manifest output/AndroidManifest.xml -A  res.apk --java ./

-o：指定链接的资源 APK 的输出路径
-I：指定 android.jar 路径
--manifest：指定 AndroidManifest.xml 路径
--java ：指定要在其中生成 R.java 的目录
```

#### 2. 所有的 AIDL 接口转化为 Java 接口

使用 AIDL（Android Interface Denifition Language），位于 sdk\build-tools 目录下的 aidl 工具，将源码文件、aidl 文件、framework.aidl 等所有的 AIDL 文件，生成相应的 Java 文件，命令如下：

```
aidl -Iaidl -pAndroid/Sdk/platforms/android-29/framework.aidl -obuild aidl/com/android/vending/billing/IInAppBillingService.aidl

-I 指定 import 语句的搜索路径，注意 -I 与目录之间一定不要有空格
-p 指定系统类的 import 语句路径，如果是要用到 android.os.Bundle 系统的类，一定要设置 sdk 的 framework.aidl 路径
-o 生成 java 文件的目录，注意 -o 与目录之间一定不要有空格，而且这设置项一定要在 aidl 文件路径之前设置
```

#### 3. 将 Java 代码编译成 Class 文件

使用 Javac（Java Compiler）把项目中所有的 Java 代码编译成 class 文件, 包括 Java 源文件、AAPT 生成的 R.java 文件 以及 aidl 生成的 Java 接口文件，命令如下：<br/>

```
javac -target 1.8 -bootclasspath platforms/android-28/android.jar -d ./java/com/testjni/*.java
```

#### 4. 将 Class 文件编译成 Dex 文件

使用 DX 工具将所有的 Class 文件（包括第三方库中的 class 文件）转换成 Dex 文件（Dalvik 可执行文件，其中包括在 Android 设备上运行的字节码），该过程主要完成 Java 字节码转换成 Dalvik 字节码, 命令如下：<br/>

```
java -jar dx.jar --dex --ouput=classes.dex ./java/com/testjni/*.class

--dex：将 class 文件转成dex文件
--output：指定生成 dex 文件到具体位置
```

#### 5. 打包生成 APK 文件 

使用 Apkbuilder（主要用到的是 sdk/tools/lib/sdklib.jar 文件中的 ApkBuilderMain 类）将所有的 Dex 文件、Resource.arsc、Res 文件夹、Assets 文件夹、AndroidManifest.xml 打包生成 APK 文件（未签名）

#### 6. 对 APK 文件签名

使用 Apksigner（Android官方针对 APK 签名及验证工具）或 Jarsigner（JDK提供针对 jar 包签名工具）对未签名的 APK 文件进行签名<br/>

**ps：如果使用 Apksigner 签名需要（7. 优化 APK 文件）放到（6. 对 APK 文件签名）签名前面，为什么？请查看关于 Apksigner 和 Jarsigner 的区别，请移步到文末**

#### 7. 优化 APK 文件

使用 zipalign 对签名后的 APK 文件进行对齐处理，对齐的主要过程是将 APK 包中所有的资源文件距离文件起始偏移为 4 字节整数倍，这样通过内存映射访问 APK 文件时的速度会更快，减少其在设备上运行时所占用的内存

#### 总结

上述打包过程都是 AndroidStudio 编译时，调用各种编译命令自动完成的, 总结一下上述打包过程：

1. 除了 assets 和 res/raw 资源被原装不动地打包进 APK 之外，其它的资源都会被编译或者处理
2. 除了 assets 资源之外，其它的资源都会被赋予一个资源 ID
3. 打包工具负责编译和打包资源，编译完成之后，会生成一个 resources.arsc 文件和一个 R.java，前者保存的是一个资源索引表，后者定义了各个资源 ID 常量
4. 应用程序配置文件 AndroidManifest.xml 同样会被编译成二进制的 xml 文件，然后再打包到 APK 里面去
5. 应用程序在运行时通过 AssetManager 来访问资源，或通过资源 ID 来访问，或通过文件名来访问<br/>

APK 文件大概可以分为两个部分：代码和资源, 代码部分通过 Javac 将 Java 代码编译成 Class 文件, 然后通过 DX 工具将 Class 文件编译成 Dex 文件，接下来我们主要来分析一下资源的编译和打包<br/>

## 资源的编译和打包

在分析资源的编译和打包之前，我们需要了解一下 Android 都有哪些资源，其实 Android 资源大概分为两个部分：assets 和 res

当我们使用 AAPT 对资源进行编译的时候，会采用两种模式 Deflate(压缩模式)/Stored(存储模式)，而具体使用模式，取决于文件后缀类型，AAPT 会对以下文件后缀类型的资源采用存储模式（即不会被压缩）

```
/* these formats are already compressed, or don't compress well */
static const char* kNoCompressExt[] = {
    ".jpg", ".jpeg", ".png", ".gif",
    ".wav", ".mp2", ".mp3", ".ogg", ".aac",
    ".mpg", ".mpeg", ".mid", ".midi", ".smf", ".jet",
    ".rtttl", ".imy", ".xmf", ".mp4", ".m4a",
    ".m4v", ".3gp", ".3gpp", ".3g2", ".3gpp2",
    ".amr", ".awb", ".wma", ".wmv", ".webm", ".mkv"
};
```

通过 `aapt l -v xxx.apk` 或 `unzip -l xxx.apk` 来查看 APK 内文件使用的什么模式

#### 1. assets 资源

assets 资源放在 assets 目录下，它里面保存一些原始的文件，可以以任何方式来进行组织，AAPT 会对指定文件后缀类型的资源进行压缩，其余的文件最终会原封不动的被打包进 APK 文件中，通过 AssetManager 来获取 asset 资源，代码如下

```
AssetManager assetManager = context.getAssets();
InputStream is = assetManager.open("fileName");
```

#### 2. res 资源

res 资源放在主工程的 res 目录下，这类资源一般都会在编译阶段生成一个资源ID供我们使用，res 目录包括 animator、anim、 color、drawable、layout、menu、raw、values、xml 等<br/>

上述资源文件除了 raw 类型资源，以及 drawable 文件夹下的 Bitmap 资源之外，其它的资源文件均会被编译成二进制格式的 XML 文件，生成的二进制格式的 XML 文件分别有一个字符串资源池，用来保存文件中引用到的每一个字符串<br/>

这样原来在文本格式的 XML 文件中的每一个放置字符串的地方在二进制格式的XML文件中都被替换成一个索引到字符串资源池的整数值，将整数值保存在 R.java 类中，R.java 会和其他源文件一起编译到 APK 中去<br/>

将资源编译成二进制文件，都是由 AAPT 工具来完成的，资源打包主要有以下几个流程：

1. 解析 AndroidManifest.xml，获得应用程序的包名称，创建资源表
2. 添加被引用资源包，被添加的资源会以一种资源 ID 的方式定义在 R.java 中
3. 资源打包工具创建一个 AaptAssets 对象，收集当前需要编译的资源文件，收集到的资源保存在 AaptAssets 对象对象中
4. 将上一步 AaptAssets 对象保存的资源，添加到资源表 ResourceTable 中去，用于最终生成资源描述文件 resources.arsc
5. 编译 values 类资源，这类资源包括数组、颜色、尺寸、字符串等值
6. 给 style、array 这类资源分配资源 ID
7. 编译 XML 资源文件，编译的流程分为：① 解析 XML 文件 ② 赋予属性名称资源 ID ③ 解析属性值 ④ 将 XML 文件从文本格式转换为二进制格式
8. 生成资源索引表 resources.arsc

##### 2.1 资源 ID

AAP 工具会所有的资源都会生成一个 R.java 文件，并且每个资源都对应 R.java 中的十六进制整数变量，其实这些十六进制的整数是由三部分组成：PackageId + TypeId + ItemValue，代码所示：

```
public final class R {
     public static final class anim {
        public static final int abc_fade_in=0x7f010000;
        public static final int abc_fade_in=0x7f010001;
        //***
     }
     public static final class string {
        public static final int a11y_no_data=0x7f100000;
        public static final int a11y_no_permission=0x7f100001;
        //***
     }
}
```

![](http://cdn.51git.cn/2020-02-13-15816057214994.jpg)

最高字节是 Package ID 表示命名空间，标明资源的来源，Android 系统自己定义了两个 Package ID，系统资源命名空间：0x01 和 应用资源命名空间：0x7f<br/>

正因为应用资源命名空间：0x7f，我们在做插件化的时候就会出现一个问题，宿主和插件包，合并资源后资源 ID 冲突。通过上面分析要解决这个问题，就要为不同的插件设置不同的 PackageId，而宿主可以保留原来 0x7f 不变，这样就永远不会有冲突发生了

**如何解决资源冲突**

1. 制定一个不用冲突的命名规范
2. library Module 的 build.gradle 中设置资源前缀(推荐)

```
android {
  resourcePrefix "<前缀>"
}
``` 

##### 2.2 资源索引(resources.arsc)

最终生成的是资源索引表 resources.arsc ，resources.arsc 是一个编译后的二进制文件, 在 AndroidStudio 打开 resources.arsc 文件，如下所示

![](http://cdn.51git.cn/2020-02-10-15813201001043.jpg)

Android 正是利用这个索引表根据资源 ID 进行资源的查找，为不同语言、不同地区、不同设备提供相对应的最佳资源。查找和通过 Resources 和 AssetManger 来完成的<br/>

在文中提到了两个工具 Apksigner 和 Jarsigner，下面一起来了解一下 Apksigner 和 Jarsigner 的区别

## Apksigner 和 Jarsigner 的区别

在 Android Studio 中点击菜单 Build->Generate signed apk... 打包签名过程中,可以看到两种签名选项 V1(Jar Signature) 和 V2(Full APK Signature) <br/>

* Jarsigner 是 JDK 提供的针对 JAR 包签名的通用工具
* Apksigner 是 Google 官方提供的针对 Android APK 签名及验证的专用工具

在 Android 11 以上使用 V4 签名，Android 9.0 以上使用 V3 签名，Android 7.0 开始使用 V2 签名，但在 Android 7.0 以下版本, 只能用旧签名方案 V1 签名

**V1 签名:**

Android 7 以下使用 V1 签名，V1 签名会对 ZIP 压缩包的每个文件进行验证, 签名后还能对压缩包修改(移动/重新压缩文件)，对 V1 签名的 APK/JAR 解压,在 META-INF 存放签名文件(MANIFEST.MF, CERT.SF, CERT.RSA), 其中 MANIFEST.MF 文件保存所有文件的 SHA1 指纹(除了 META-INF 文件), 由此可知: V1 签名是对压缩包中单个文件签名验证


**V2 签名:**

Android 7 开始增加了 V2 签名，V2 签名会对 ZIP 压缩包的整个文件验证, 签名后不能修改压缩包(包括 zipalign), 对 V2 签名的 APK 解压, 没有发现签名文件, 重新压缩后 V2 签名就失效, 由此可知: V2 签名是对整个 APK 签名验证
   
**V3 签名:**

Android 9 增加了 V3 签名，V3 签名在 V2 的基础上，仍然采用检查整个压缩包的校验方式，支持 APK 密钥轮替，这使应用能够在 APK 更新过程中更改其签名密钥

v3 签名新增的新块（attr） 会记录我们之前的签名信息以及新的签名信息，支持 APK 密钥轮替方案，来做签名的替换和升级。这意味着，只要旧签名证书在手，我们就可以通过它在新的 APK 文件中，更改签名。
    
需要注意的是：对于覆盖安装的情况，签名校验只支持升级，而不支持降级

**V4 签名:**

在 Android 11 之前，建议不要使用 APK 密钥轮替，在 Android 11 之后增加了 V4 签名，V4 签名将签名存储在单独的 <apk name>.apk.idsig 文件中。v4 签名需要 v2 或 v3 签名作为补充。

关于签名更多内容，会在后续文章内介绍。

**总结:**

* V1 签名是对压缩包中单个文件签名验证
* V2 签名是对整个 APK 签名验证
* zipalign 可以在 V1 签名后执行
* zipalign 不能在 V2 签名后执行,只能在 V2 签名之前执行
* V2 签名更安全(不能修改压缩包)
* V2 签名验证时间更短(不需要解压验证), 因而安装速度加快
* apksigner 工具默认同时使用 V1 和 V2 签名, 以兼容 Android 7.0 以下版本
* Android 7 以下使用 V1 签名
* Android 7 开始增加了 V2 签名  
* Android 9 增加了 V3 签名
* Android 11 之后增加了 V4 签名
* V3 签名 和 V4 签名 目前只能在 Google Play 上使用

## 参考文献

* [Google的 Apk 构建流程](https://developer.android.com/studio/build/index.html?hl=zh-cn#build-process)
* [Android Studio 中为应用签名](https://developer.android.com/studio/build/index.html?hl=zh-cn#build-process)
* [AAPT2](https://developer.android.com/studio/command-line/aapt2?hl=zh-cn)

## 结语

致力于分享一系列 Android 系统源码、逆向分析、算法、翻译、Jetpack  源码相关的文章，如果你同我一样喜欢研究 Android 源码，可以关注我，如果你喜欢这篇文章欢迎 star，一起来学习，期待与你一起成长

**算法**

由于 LeetCode 的题库庞大，每个分类都能筛选出数百道题，由于每个人的精力有限，不可能刷完所有题目，因此我按照经典类型题目去分类、和题目的难易程度去排序

* 数据结构： 数组、栈、队列、字符串、链表、树……
* 算法： 查找算法、搜索算法、位运算、排序、数学、……

每道题目都会用 Java 和 kotlin 去实现，并且每道题目都有解题思路，如果你同我一样喜欢算法、LeetCode，可以关注我 GitHub 上的 LeetCode 题解：[Leetcode-Solutions-with-Java-And-Kotlin](https://github.com/hi-dhl/Leetcode-Solutions-with-Java-And-Kotlin)，一起来学习，期待与你一起成长




