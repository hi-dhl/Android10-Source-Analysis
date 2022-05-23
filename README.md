#  <p align="center"> Android 10 Source Analysis </p>

<p align="center">仓库状态：持续更新中，会优先发布到公众号</p>

<p align="center"> 致力于分享一系列 Android 系统源码，如果你同我一样喜欢研究 Android 源码，一起来学习，期待与你一起成长 </p>

<p align="center">
<a href="https://github.com/hi-dhl"><img src="https://img.shields.io/badge/GitHub-HiDhl-4BC51D.svg?style=flat"></a> <img src="https://img.shields.io/badge/language-Java | Kotlin-orange.svg"/> <img src="https://img.shields.io/badge/platform-android-lightgrey.svg"/>
</p>

![Android10](http://cdn.51git.cn/2020-06-04-15911987213302.jpg)

**仓库状态：持续更新中，会优先发布到公众号**

### 联系我
 
<div align="center">
    <table>
        <tr>
             <td><a href="https://juejin.im/user/2594503168898744">掘金</a></td>
             <td><a href="https://hi-dhl.com">博客</td>
             <td><a href="https://github.com/hi-dhl">Github</a></td>
             <td><a href="https://www.zhihu.com/people/hi-dhl">知乎</a></td>
             <td><a href="https://space.bilibili.com/498153238">哔哩哔哩</a></td>
             <td><a href="https://site.51git.cn">互联网人专属导航站</a></td>
         </tr>
     </table>
</div>

* 个人微信：hi-dhl
* 公众号：ByteCode，包含 Jetpack ，Kotlin ，Android 10 系列源码，译文，LeetCode / 剑指 Offer / 多线程 / 国内外大厂算法题 等等一系列文章

![](https://img.hi-dhl.com/ercode3.png)


### 代码版本

分支：android-10.0.0_r14

Android 是一个非常庞大的系统，了解系统源码，不仅有助于分析问题，在面试过程中，对我们也是非常有帮助的

### 为什么要这件事情

* 市面上大部分的源码分析都是基于 6.0、7.0、8.0 等等，10.0 之后源码变化还是挺大的
* 很多书籍和博客，文章中罗列了大量的代码，很难有耐心深入的阅读下去，本系列文章中没有大量的代码，采用图文并茂、表格汇总的方式，进行分析、总结、归纳
* 之前也分析过其他版本的 Android 源码，但是比较零散，主要偏向工作过程中自己负责的相关部分的源码
* 是对自己各方面能力的提高，因为自己看懂只是输入，但是输出的过程，会强迫自己查阅很多官方资料、总结、分析，输出成一篇完整的文章，可以学习和了解到更深层次的内容

致力于分享一系列 Android 系统源码、逆向分析、算法、翻译、Jetpack  源码相关的文章，如果你同我一样喜欢研究 Android 源码，可以关注我，一起来学习，期待与你一起成长

### 文章目录

* [0xA01 Android 10 源码分析：APK 是如何生成的](https://github.com/hi-dhl/Android10-Source-Analysis/blob/master/0xA01%20Android%2010%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%EF%BC%9AAPK%20%E6%98%AF%E5%A6%82%E4%BD%95%E7%94%9F%E6%88%90%E7%9A%84.md)
* [0xA02 Android 10 源码分析：APK 的安装流程](https://github.com/hi-dhl/Android10-Source-Analysis/blob/master/0xA02%20Android%2010%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%EF%BC%9AAPK%20%E7%9A%84%E5%AE%89%E8%A3%85%E6%B5%81%E7%A8%8B.md)
* [0xA03 Android 10 源码分析：APK 加载流程之资源加载](https://github.com/hi-dhl/Android10-Source-Analysis/blob/master/0xA03%20Android%2010%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%EF%BC%9AAPK%20%E5%8A%A0%E8%BD%BD%E6%B5%81%E7%A8%8B%E4%B9%8B%E8%B5%84%E6%BA%90%E5%8A%A0%E8%BD%BD.md)
* [0xA04 Android 10 源码分析：Apk加载流程之资源加载（二）](https://github.com/hi-dhl/Android10-Source-Analysis/blob/master/0xA04%20Android%2010%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%EF%BC%9AApk%E5%8A%A0%E8%BD%BD%E6%B5%81%E7%A8%8B%E4%B9%8B%E8%B5%84%E6%BA%90%E5%8A%A0%E8%BD%BD%EF%BC%88%E4%BA%8C%EF%BC%89.md)
* [0xA05 Android 10 源码分析：Dialog加载绘制流程以及在Kotlin、DataBinding中的使用](https://github.com/hi-dhl/Android10-Source-Analysis/blob/master/0xA05%20Android%2010%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%EF%BC%9ADialog%E5%8A%A0%E8%BD%BD%E7%BB%98%E5%88%B6%E6%B5%81%E7%A8%8B%E4%BB%A5%E5%8F%8A%E5%9C%A8Kotlin%E3%80%81DataBinding%E4%B8%AD%E7%9A%84%E4%BD%BF%E7%94%A8.md)
* [0xA06 Android 10 源码分析：WindowManager 视图绑定以及体系结构](https://github.com/hi-dhl/Android10-Source-Analysis/blob/master/0xA06%20Android%2010%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%EF%BC%9AWindowManager%20%E8%A7%86%E5%9B%BE%E7%BB%91%E5%AE%9A%E4%BB%A5%E5%8F%8A%E4%BD%93%E7%B3%BB%E7%BB%93%E6%9E%84.md)
* [0xA07 Android 10 源码分析：Window 的类型 以及 三维视图层级分析](https://github.com/hi-dhl/Android10-Source-Analysis/blob/master/0xA07%20Android%2010%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%EF%BC%9AWindow%20%E7%9A%84%E7%B1%BB%E5%9E%8B%20%E4%BB%A5%E5%8F%8A%20%E4%B8%89%E7%BB%B4%E8%A7%86%E5%9B%BE%E5%B1%82%E7%BA%A7%E5%88%86%E6%9E%90.md)
* 持续更新中......

> 正在建立一个最全、最新的 AndroidX Jetpack 相关组件的实战项目 以及 相关组件原理分析文章，目前已经包含了 App Startup、Paging3、Hilt 等等，正在逐渐增加其他 Jetpack 新成员，仓库持续更新，可以前去查看：[AndroidX-Jetpack-Practice](https://github.com/hi-dhl/AndroidX-Jetpack-Practice)。


另外我还在做另外一件事情，算法题库的归纳和总结，在大学期间经常参加一些比赛如蓝桥杯、ACM 等等，因此无论在面试还是工作都带来很多帮助，知道数据结构和算法的重要性，也是面试的入门门槛之一

### 算法题库的归纳和总结

由于 LeetCode 的题库庞大，每个分类都能筛选出数百道题，由于每个人的精力有限，不可能刷完所有题目，因此我按照经典类型题目去分类、和题目的难易程度去排序

* 数据结构： 数组、栈、队列、字符串、链表、树……
* 算法： 查找算法、搜索算法、位运算、排序、数学、……

每道题目都会用 Java 和 kotlin 去实现，并且每道题目都有解题思路，如果你同我一样喜欢算法、LeetCode，可以关注我 GitHub 上的 LeetCode 题解：[Leetcode-Solutions-with-Java-And-Kotlin](https://github.com/hi-dhl/Leetcode-Solutions-with-Java-And-Kotlin)，一起来学习，期待与你一起成长

### Android10 源码分析

正在写一系列的 Android 10 源码分析的文章，了解系统源码，不仅有助于分析问题，在面试过程中，对我们也是非常有帮助的，如果你同我一样喜欢研究 Android 源码，可以关注我 GitHub 上的 [Android10-Source-Analysis](https://github.com/hi-dhl/Android10-Source-Analysis)。

### 精选国外的技术文章

目前正在整理和翻译一系列精选国外的技术文章，不仅仅是翻译，很多优秀的英文技术文章提供了很好思路和方法，每篇文章都会有**译者思考**部分，对原文的更加深入的解读，可以关注我 GitHub 上的 [Technical-Article-Translation](https://github.com/hi-dhl/Technical-Article-Translation)。

