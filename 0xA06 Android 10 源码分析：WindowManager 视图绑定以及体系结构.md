# 0xA06 Android 10 源码分析：WindowManager 视图绑定以及体系结构

![WindowManager-Vie](http://cdn.51git.cn/2020-05-03-WindowManager-View.png)

## 引言

> * 这是 Android 10 源码分析系列的第 6 篇
> * 分支：android-10.0.0_r14
> * 全文阅读大概 10 分钟

**通过这篇文章你将学习到以下内容，将在文末总结部分会给出相应的答案**

* Acivity 和 Dialog 视图解析绑定过程？
* Activity 的视图如何与 Window 关联的？
* Window 如何与 WindowManager 关联？
* Dialog 的视图如何与 Window 关联？

本文主要分析 Activity、Window、PhoneWindow、WindowManager 之间的关系，为我们后面的文章 **「如何在 Andorid 系统里添加自定义View」** 等等文章奠定基础，先来了解一下它们的基本概念

![windo](http://cdn.51git.cn/2020-05-02-window.png)

* Activity：应用视图的容器。
* WindowManager：它是一个接口类，继承自接口 ViewManager，对 Window 进行管理
* Window：它是一个抽象类，它作为一个顶级视图添加到 WindowManager 中，对 View 进行管理
* PhoneWindow：Window唯一实现类，Window是一个抽象概念，添加到WindowManager的根容器
* DecorView: 它是 PhoneWindow 内部的一个成员变量，继承自 FrameLayout，FrameLayout 继承自 ViewGroup

在分析他们之前的关系之前，我们先来回顾一下 Acivity 和 Dialog 视图解析绑定的过程

## Acivity 和 Dialog 视图解析绑定的过程

Acivity 和 Dialog 相关的文章：

* [0xA03 Android 10 源码分析：APK 加载流程之资源加载](https://juejin.im/post/5e6c8c14f265da574b792a1a) 
* [0xA04 Android 10 源码分析：APK 加载流程之资源加载（二）](https://juejin.im/post/5e7f0f2c51882573c4676bc7) 
* [0xA05 Android 10 源码分析：Dialog 加载绘制流程以及在 Kotlin、DataBinding 中的使用](https://juejin.im/post/5e9199db6fb9a03c7916f635)

在之前的文章 分别介绍了 Acivity 和 自定义 Dialog 视图的解析和绑定，总的来说分为三步

* 1. 调用 LayoutInflater 的 inflate 方法，深度优先遍历解析 View
* 2. 调用 ViewGroup 的 addView 方法将子 View 添加到根布局中
* 3. 调用 WindowManager 的 addView 方法添加根布局

LayoutInflater 的 inflate 方法有多个重载的方法，常用的是下面三个参数的方法
**frameworks/base/core/java/android/view/LayoutInflater.java**

```
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
    return inflate(resource, root, root != null);
}
```

* resource：要解析的 xml 布局文件 Id
* root：表示根布局
* attachToRoot：是否要添加到父布局 root 中

resource 其实很好理解就是资源 Id，而 root 和 attachToRoot 分别代表什么意思：

* 当 attachToRoot == true 且 root ！= null 时，新解析出来的 View 会被 add 到 root 中去，然后将 root 作为结果返回
* 当 attachToRoot == false 且 root ！= null 时，新解析的 View 会直接作为结果返回，而且 root 会为新解析的 View 生成 LayoutParams 并设置到该 View 中去
* 当 attachToRoot == false 且 root == null 时，新解析的 View 会直接作为结果返回

当 View 解析完成之后，最后会调用 WindowManager 的 addView 方法，WindowManager 是一个接口类，继承自接口 ViewManager，用来管理 Window，它的实现类为 WindowManagerImpl，所以调用 WindowManager 的 addView 方法，实际上调用的是 WindowManagerImpl 的 addView 方法
**frameworks/base/core/java/android/view/WindowManagerImpl.java**

```
public final class WindowManagerImpl implements WindowManager {
    @UnsupportedAppUsage
    // 单例的设计模式
    private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
    private final Context mContext;
    private final Window mParentWindow;
    ......
    
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        // mGlobal 是 WindowManagerGlobal 的实例
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }
    ......
    
}
```

mGlobal 是 WindowManagerGlobal 的实例，使用的单例设计模式，参数 mParentWindow 是 Window 的实例，实际上是委托给 WindowManagerGlobal 去实现的 <br/>

到这里我们关于 Acivity 和 Dialog 视图的解析和添加过程大概介绍完了，关于 Dialog 的视图如何与 Window 绑定在 [0xA05 Android 10 源码分析：Dialog 加载绘制流程以及在 Kotlin、DataBinding 中的使用](https://juejin.im/post/5e9199db6fb9a03c7916f635) 文章中介绍了，接下来分析一下 Activity、Window、WindowManager 的关系

## Activity、Window、WindowManager 的关系

在 Activity 内部维护着一个 Window 的实例变量 mWindow
**frameworks/base/core/java/android/app/Activity.java**

```
public class Activity extends ContextThemeWrappe{
  private Window mWindow;
}
``` 

Window 是一个抽象类，它的具体实现类为 PhoneWindow，在 Activity 的 attach 方法中给 Window 的实例变量 mWindow 赋值
**frameworks/base/core/java/android/app/Activity.java**

```
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor,
        Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken) {
    ......

    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    ......

    mWindow.setWindowManager(
            (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
            mToken, mComponent.flattenToString(),
            (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    ......
    
}
```

* 创建了 PhoneWindow 并赋值给 mWindow
* 调用 PhoneWindow 的 setWindowManager 方法，这个方法的具体实现发生在 Window 中，最终调用的是 Window 的 setWindowManager 方法
**frameworks/base/core/java/android/view/Window.java**

```
public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
        boolean hardwareAccelerated) {
    ......
    // mWindowManager 是 WindowManagerImpl的实例变量
    mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
}
```

将 WindowManager 转换为 WindowManagerImpl，之后调用 createLocalWindowManager 方法，并传递当前的 Window 对象，构建 WindowManagerImpl 对象，之后赋值给 mWindowManager
**frameworks/base/core/java/android/view/WindowManagerImpl.java**

```
public WindowManagerImpl createLocalWindowManager(Window parentWindow) {
    return new WindowManagerImpl(mContext, parentWindow);
}
```

其实在 createLocalWindowManager 方法中，就做了一件事，将 Window 作为参数构建了一个 WindowManagerImpl 对象返还给调用处<br/>

总的来说，其实就是在 Activity 的 attach 方法中，通过调用 Window 的 setWindowManager 方法将 Window 和 WindowManager 关联在了一起<br/>

PhoneWindow 是 Window 的实现类，它是一个窗口，本身并不具备 View 相关的能力，实际上在 PhoneWindow 内部维护这一个变量 mDecor
**frameworks/base/core/java/com/android/internal/policy/PhoneWindow.java**

```
public class PhoneWindow extends Window{
  // This is the top-level view of the window, containing the   window decor.
  private DecorView mDecor; 
  
  private void installDecor() {
        mForceDecorInstall = false;
        if (mDecor == null) {
            // 完成DecorView的实例化
            mDecor = generateDecor(-1);
            ......
        }
        
        if (mContentParent == null) {
            // 调用 generateLayout 方法 要负责了DecorView的初始设置，诸如主题相关的feature、DecorView的背景
            mContentParent = generateLayout(mDecor);
        } 
        ......
    }
    
    // 完成DecorView的实例化
    protected DecorView generateDecor(int featureId) {
        ......
        return new DecorView(context, featureId, this, getAttributes());
    }
    
    // 调用 generateLayout 方法 要负责了DecorView的初始设置，
    // 诸如主题相关的feature、DecorView的背景，同时也初始化 contentParent
    protected ViewGroup generateLayout(DecorView decor) {
        ......
        
        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
        
        ......
    }

}
```

* mDecor 是 window 的顶级视图，它继承自 FrameLayout，它的创建过程由 installDecor 完成，然后在 installDecor 方法中通过 generateDecor 方法来完成DecorView的实例化
* 调用 generateLayout 方法 要负责了DecorView的初始设置，诸如主题相关的feature、DecorView的背景，同时也初始化 contentParent
* mDecor 它实际上是一个 ViewGroup，当在 Activity 中调用 setContentView 方法，通过调用 inflater 方法把布局资源转换为一个 View，然后添加到 DecorView 的 mContenParnent 中

当 View 初始化完成之后，最后会进入 ActivityThread 的 handlerResumeActivity 方法，执行了r.activity.makeVisible()方法
**frameworks/base/core/java/android/app/ActivityThread.java**

```
public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
            String reason) {
        ......
        
        if (r.activity.mVisibleFromClient) {
            r.activity.makeVisible();
        }
        ......
    }
```

最终调用 Activity 的 makeVisible 方法，把 decorView 添加到 WindowManage 中
**frameworks/base/core/java/android/app/Activity.java**

```
void makeVisible() {
    if (!mWindowAdded) {
        ViewManager wm = getWindowManager();
        wm.addView(mDecor, getWindow().getAttributes());
        mWindowAdded = true;
    }
    mDecor.setVisibility(View.VISIBLE);
}
```

到这里他们之间的关系明确了：

* 一个 Activity 持有一个 PhoneWindow 的对象，而一个 PhoneWindow 对象持有一个 DecorView 的实例
* PhoneWindow 继承自 Window，一个 Window 对象内部持有 mWindowManager 的实例，通过调用 setWindowManager 方法与 WindowManager 关联在一起
* WindowManager 继承自 ViewManager，WindowManagerImpl 是 WindowManager 接口的实现类，但是具体的功能都会委托给 WindowManagerGlobal 来实现
* 调用 WindowManager 的 addView 方法，实际上调用的是 WindowManagerImpl 的 addView 方法

## 总结

#### Acivity 和 Dialog 视图解析绑定过程？

* 1. 调用 LayoutInflater 的 inflate 方法，深度优先遍历解析 View
* 2. 调用 ViewGroup 的 addView 方法将子 View 添加到根布局中
* 3. 调用 WindowManager 的 addView 方法添加根布局


#### Activity 的视图如何与 Window 关联的？

* 在 Activity 内部维护着一个 Window 的实例变量 mWindow
**frameworks/base/core/java/android/app/Activity.java**

```
public class Activity extends ContextThemeWrappe{
  private Window mWindow;
}
``` 

* 最后调用 Activity 的 makeVisible 方法，把 decorView 添加到 WindowManage 中
**frameworks/base/core/java/android/app/Activity.java**

```
void makeVisible() {
    if (!mWindowAdded) {
        ViewManager wm = getWindowManager();
        wm.addView(mDecor, getWindow().getAttributes());
        mWindowAdded = true;
    }
    mDecor.setVisibility(View.VISIBLE);
}
```

#### Window 如何与 WindowManager 关联？

在 Activity 的 attach 方法中，调用 PhoneWindow 的 setWindowManager 方法，这个方法的具体实现发生在 Window 中，最终调用的是 Window 的 setWindowManager 方法，将 Window 和 WindowManager 关联在了一起
**frameworks/base/core/java/android/view/Window.java**

```
public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
        boolean hardwareAccelerated) {
    ......
    // mWindowManager 是 WindowManagerImpl的实例变量
    mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
}
```

#### Dialog 的视图如何与 Window 关联？

* 在 Dialog 的构造方法中初始化了 Window 对象
 **frameworks/base/core/java/android/app/Dialog.java**

```
Dialog(@NonNull Context context, @StyleRes int themeResId, boolean createContextThemeWrapper) {
    ...
    // 获取WindowManager对象
    mWindowManager = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
    // 构建PhoneWindow
    final Window w = new PhoneWindow(mContext);
    // mWindow 是PhoneWindow实例
    mWindow = w;
   ...
}
```

* 调用 Dialog 的 show 方法，完成 view 的绘制和 Dialog 的显示
**frameworks/base/core/java/android/app/Dialog.java**

```
public void show() {
    // 获取DecorView
    mDecor = mWindow.getDecorView();
    // 获取布局参数
    WindowManager.LayoutParams l = mWindow.getAttributes();
    // 将DecorView和布局参数添加到WindowManager中
    mWindowManager.addView(mDecor, l);
}
```

## 参考文献

* [http://liuwangshu.cn/framework/wm/1-windowmanager.html](http://liuwangshu.cn/framework/wm/1-windowmanager.html)
* [https://gudong.site/2017/05/08/activity-windown-decorview.html](https://gudong.site/2017/05/08/activity-windown-decorview.html)

## 结语

致力于分享一系列 Android 系统源码、逆向分析、算法、翻译、Jetpack  源码相关的文章，如果你同我一样喜欢研究 Android 源码，可以关注我，如果你喜欢这篇文章欢迎 star，一起来学习，期待与你一起成长

**算法**

由于 LeetCode 的题库庞大，每个分类都能筛选出数百道题，由于每个人的精力有限，不可能刷完所有题目，因此我按照经典类型题目去分类、和题目的难易程度去排序

* 数据结构： 数组、栈、队列、字符串、链表、树……
* 算法： 查找算法、搜索算法、位运算、排序、数学、……

每道题目都会用 Java 和 kotlin 去实现，并且每道题目都有解题思路，如果你同我一样喜欢算法、LeetCode，可以关注我 GitHub 上的 LeetCode 题解：[Leetcode-Solutions-with-Java-And-Kotlin](https://github.com/hi-dhl/Leetcode-Solutions-with-Java-And-Kotlin)，一起来学习，期待与你一起成长


