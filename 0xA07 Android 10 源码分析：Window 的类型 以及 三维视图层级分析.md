# 0xA07 Android 10 源码分析：Window 的类型 以及 三维视图层级分析

![Window视图层级](http://cdn.51git.cn/2020-06-03-Window视图层级.png)

## 引言

> * 这是 Android 10 源码分析系列的第 7 篇
> * 分支：android-10.0.0_r14
> * 全文阅读大概 10 分钟

在之前的文章 [0xA06 Android 10 源码分析：WindowManager 视图绑定以及体系结构](https://juejin.im/post/5ead0b865188256d545fd2f8) 介绍了  Activity、Window、PhoneWindow、WindowManager 之间的关系，以及 Activity 和 Dialog 的视图绑定过程，而这篇文章主要两个目的：

1. 对上一篇文章 [0xA06 Android 10 源码分析：WindowManager 视图绑定以及体系结构](https://juejin.im/post/5ead0b865188256d545fd2f8) 做深入的了解
2. 为后面的篇文章「如何在 Andorid 系统里添加自定义 View」等等做好铺垫

**通过这篇文章你将学习到以下内容，将在文末总结部分会给出相应的答案**

* Window 都有那些常用的参数?
* Window 都那些类型？每个类型的意思？以及作用？
* Window 那些过时的 API 以及处理方案？
* Window 视图层级顺序是如何确定的？
* Window 都那些 flag？每个 flag 的意思？以及作用？
* Window 的软键盘模式？每个模式的意思？以及如何使用？
* Kotlin 小技巧？

在开始分析之前，我们先来看一张图，熟悉一下几个基本概念，这些概念伴将随着整篇文章

![Window 视图层级顺序](http://cdn.51git.cn/2020-06-03-640.gif)

* 我们在手机上看到的界面是二维的，但是实际上是一个三维，如上图所示
* Window：是一个抽象类，它作为一个顶级视图添加到 WindowManager 中，View 是依附于 Window 而存在的，对 View 进行管理
* WindowManager：它是一个接口，继承自接口 ViewManager，对 Window 进行管理
* PhoneWindow：Window 唯一实现类，添加到 WindowManager 的根容器中
* WindowManagerService：WindowManager 是 Window 的容器，管理着 Window，对 Window 进行添加和删除，最终具体的工作都是由 WindowManagerService 来处理的，WindowManager 和 WindowManagerService 通过 Binder 来进行跨进程通信，WindowManagerService 才是 Window 的最终管理者

这篇文章重要知识点是 **Window 视图层级顺序是如何确定的**，其他内容都是一些概念的东西，可以选择性的阅读，了解完基本概念之后，进入这篇文章的核心内容，我们先来了解一下 Window 都有那些常用的参数

## Window 都有那些常用的参数

Window 的参数都被定义在 WindowManager 的静态内部类 LayoutParams 中
**frameworks/base/core/java/android/view/WindowManager#LayoutParams.java**

```
public static class LayoutParams extends ViewGroup.LayoutParams implements Parcelable {
    // window 左上角的 x 坐标
    public int x;
    
    // window 左上角的 y 坐标
    public int y;
    
    // Window 的类型
    public int type;
    
    // Window 的 flag 用于控制 Window 的显示
    public int flags;
    
    // window 软键盘输入区域的显示模式
    public int softInputMode;
    
    // window 的透明度，取值为0-1
    public float alpha = 1.0f;
    
    // window 在屏幕中的位置
    public int gravity;
    
    // window 的像素点格式，值定义在 PixelFormat 中
    public int format;
}
```

接下来我们我们主要来介绍一下 Window 的类型、Window 视图层级顺序、Window 的 flag、和 window 软键盘模式

### Window 都那些类型以及作用

Window 的类型大概可以分为三类： 应用程序 Window（Application Window）、子 Window（Sub Windwow）、系统 Window（System Window）， Window 的类型通过 type 值来表示，每个大类型又包含多个小类型，它们都定义在 WindowManager 的静态内部类 LayoutParams
**frameworks/base/core/java/android/view/WindowManager#LayoutParams.java**

```
public static class LayoutParams extends ViewGroup.LayoutParams implements Parcelable {
    public int type;
    
    // 应用程序 Window 的开始值
    public static final int FIRST_APPLICATION_WINDOW = 1;
    // 应用程序 Window 的结束值
    public static final int LAST_APPLICATION_WINDOW = 99;
    
    // 子 Window 类型的开始值
    public static final int FIRST_SUB_WINDOW = 1000;
    // 子 Window 类型的结束值
    public static final int LAST_SUB_WINDOW = 1999;
    
    // 系统 Window 类型的开始值
    public static final int FIRST_SYSTEM_WINDOW     = 2000;   
    // 系统 Window 类型的结束值
    public static final int LAST_SYSTEM_WINDOW      = 2999;
}
```

| 类型 | 值 | 备注 |
| --- | --- | --- |
| FIRST_APPLICATION_WINDOW | 1 | 应用程序 Window 的开始值 |
| LAST_APPLICATION_WINDOW | 99 | 应用程序 Window 的结束值 |
| FIRST_SUB_WINDOW | 1000 | 子 Window 的开始值 |
| LAST_SUB_WINDOW | 1999 | 子 Window 的结束值 |
| FIRST_SYSTEM_WINDOW | 2000 | 系统 Window 的开始值 |
| LAST_SYSTEM_WINDOW | 2999 | 系统 Window 的结束值 |

**小技巧：如果是层级在 2000（FIRST_SYSTEM_WINDOW）以下的是不需要申请弹窗权限的**

* 应用程序 Window（Application Window）：它的区间范围 [1,99]，例如 Activity
**frameworks/base/core/java/android/view/WindowManager#LayoutParams.java**

```
// 应用程序 Window 的开始值
public static final int FIRST_APPLICATION_WINDOW = 1;

// 应用程序 Window 的基础值
public static final int TYPE_BASE_APPLICATION   = 1;

// 普通的应用程序
public static final int TYPE_APPLICATION        = 2;

// 特殊的应用程序窗口，当程序可以显示 Window 之前使用这个 Window 来显示一些东西
public static final int TYPE_APPLICATION_STARTING = 3;

// TYPE_APPLICATION 的变体，在应用程序显示之前，WindowManager 会等待这个 Window 绘制完毕
public static final int TYPE_DRAWN_APPLICATION = 4;

// 应用程序 Window 的结束值
public static final int LAST_APPLICATION_WINDOW = 99;
```

| <div style="width: 180pt">类型</div>| 备注 |
| --- | --- |
| FIRST_APPLICATION_WINDOW | 应用程序 Window 的开始值 |
| TYPE_BASE_APPLICATION | 应用程序 Window 的基础值 |
| TYPE_APPLICATION | 普通的应用程序 |
| TYPE_APPLICATION_STARTING | 特殊的应用程序窗口，当程序可以显示 Window 之前使用这个<br/> Window 来显示一些东西 |
| TYPE_DRAWN_APPLICATION | TYPE_APPLICATION 的变体 在应用程序显示之前，<br/>WindowManager 会等待这个 Window 绘制完毕 |
| LAST_APPLICATION_WINDOW | 应用程序 Window 的结束值 |


* 子 Window（Sub Windwow）：它的区间范围 [1000,1999]，这些 Window 按照 Z-order 顺序依附于父 Window 上（关于 Z-order 后文有介绍），并且他们的坐标空间相对于父 Window 的，例如：PopupWindow
**frameworks/base/core/java/android/view/WindowManager#LayoutParams.java**

```
// 子 Window 类型的开始值
public static final int FIRST_SUB_WINDOW = 1000;

// 应用程序 Window 顶部的面板。这些 Window 出现在其附加 Window 的顶部。
public static final int TYPE_APPLICATION_PANEL = FIRST_SUB_WINDOW;

// 用于显示媒体(如视频)的 Window。这些 Window 出现在其附加 Window 的后面。
public static final int TYPE_APPLICATION_MEDIA = FIRST_SUB_WINDOW + 1;

// 应用程序 Window 顶部的子面板。这些 Window 出现在其附加 Window 和任何Window的顶部
public static final int TYPE_APPLICATION_SUB_PANEL = FIRST_SUB_WINDOW + 2;

// 当前Window的布局和顶级Window布局相同时，不能作为子代的容器
public static final int TYPE_APPLICATION_ATTACHED_DIALOG = FIRST_SUB_WINDOW + 3;

// 用显示媒体 Window 覆盖顶部的 Window， 这是系统隐藏的 API
public static final int TYPE_APPLICATION_MEDIA_OVERLAY  = FIRST_SUB_WINDOW + 4;

// 子面板在应用程序Window的顶部，这些Window显示在其附加Window的顶部， 这是系统隐藏的 API
public static final int TYPE_APPLICATION_ABOVE_SUB_PANEL = FIRST_SUB_WINDOW + 5;

// 子 Window 类型的结束值
public static final int LAST_SUB_WINDOW = 1999;
```

| <div style="width: 240pt">类型 </div> | 备注 |
| --- | --- |
| FIRST_SUB_WINDOW | 子 Window 的开始值 |
| TYPE_APPLICATION_PANEL | 应用程序 Window 顶部的面板，这些 Window 出现在其附加 Window 的顶部 |
| TYPE_APPLICATION_MEDIA | 用于显示媒体(如视频)的 Window，这些 Window 出现在其附加 Window 的后面 |
| TYPE_APPLICATION_SUB_PANEL | 应用程序 Window 顶部的子面板，这些 Window 出现在其附加 Window 和任何Window的顶部 |
| TYPE_APPLICATION_ATTACHED_DIALOG | 当前Window的布局和顶级Window布局相同时，不能作为子代的容器 |
| TYPE_APPLICATION_MEDIA_OVERLAY |  用显示媒体 Window 覆盖顶部的 Window， 这是系统隐藏的 API |
| TYPE_APPLICATION_ABOVE_SUB_PANEL | 子面板在应用程序Window的顶部，这些Window显示在其附加Window的顶部， 这是系统隐藏的 API|
| LAST_SUB_WINDOW | 子 Window 的结束值 |
 
* 系统 Window（System Window）: 它区间范围 [2000,2999]，例如：Toast，输入法窗口，系统音量条窗口，系统错误窗口
**frameworks/base/core/java/android/view/WindowManager#LayoutParams.java**

```
// 系统Window类型的开始值
public static final int FIRST_SYSTEM_WINDOW     = 2000;

// 系统状态栏，只能有一个状态栏，它被放置在屏幕的顶部，所有其他窗口都向下移动
public static final int TYPE_STATUS_BAR         = FIRST_SYSTEM_WINDOW;

// 系统搜索窗口，只能有一个搜索栏，它被放置在屏幕的顶部
public static final int TYPE_SEARCH_BAR         = FIRST_SYSTEM_WINDOW+1;

@Deprecated
// API 已经过时，用 TYPE_APPLICATION_OVERLAY 代替
public static final int TYPE_PHONE              = FIRST_SYSTEM_WINDOW+2;

@Deprecated
// API 已经过时，用 TYPE_APPLICATION_OVERLAY 代替
public static final int TYPE_SYSTEM_ALERT       = FIRST_SYSTEM_WINDOW+3;

// 已经从系统中被移除，可以使用 TYPE_KEYGUARD_DIALOG 代替
public static final int TYPE_KEYGUARD           = FIRST_SYSTEM_WINDOW+4;

@Deprecated
// API 已经过时，用 TYPE_APPLICATION_OVERLAY 代替
public static final int TYPE_TOAST              = FIRST_SYSTEM_WINDOW+5;

@Deprecated
// API 已经过时，用 TYPE_APPLICATION_OVERLAY 代替
public static final int TYPE_SYSTEM_OVERLAY     = FIRST_SYSTEM_WINDOW+6;

@Deprecated
// API 已经过时，用 TYPE_APPLICATION_OVERLAY 代替
public static final int TYPE_PRIORITY_PHONE     = FIRST_SYSTEM_WINDOW+7;

// 系统对话框窗口
public static final int TYPE_SYSTEM_DIALOG      = FIRST_SYSTEM_WINDOW+8;

// 锁屏时显示的对话框
public static final int TYPE_KEYGUARD_DIALOG    = FIRST_SYSTEM_WINDOW+9;

@Deprecated
// API 已经过时，用 TYPE_APPLICATION_OVERLAY 代替
public static final int TYPE_SYSTEM_ERROR       = FIRST_SYSTEM_WINDOW+10;

// 输入法窗口，位于普通 UI 之上，应用程序可重新布局以免被此窗口覆盖
public static final int TYPE_INPUT_METHOD       = FIRST_SYSTEM_WINDOW+11;

// 输入法对话框，显示于当前输入法窗口之上
public static final int TYPE_INPUT_METHOD_DIALOG= FIRST_SYSTEM_WINDOW+12;

// 墙纸
public static final int TYPE_WALLPAPER          = FIRST_SYSTEM_WINDOW+13;

// 状态栏的滑动面板
public static final int TYPE_STATUS_BAR_PANEL   = FIRST_SYSTEM_WINDOW+14;

// 应用程序叠加窗口显示在所有窗口之上
public static final int TYPE_APPLICATION_OVERLAY = FIRST_SYSTEM_WINDOW + 38;

// 系统Window类型的结束值
public static final int LAST_SYSTEM_WINDOW      = 2999;
```

| <div style="width: 180pt">类型 </div>  | 备注 |
| --- | --- |
| FIRST_SYSTEM_WINDOW | 系统 Window 类型的开始值 |
| TYPE_STATUS_BAR | 系统状态栏，只能有一个状态栏，它被放置在屏幕的顶部，所有其他窗口都向下移动 |
| TYPE_SEARCH_BAR | 系统搜索窗口，只能有一个搜索栏，它被放置在屏幕的顶部 |
| TYPE_PHONE | API 已经过时，用 TYPE_APPLICATION_OVERLAY 代替 |
| TYPE_SYSTEM_ALERT | API 已经过时，用 TYPE_APPLICATION_OVERLAY 代替 |
| TYPE_KEYGUARD | 已经从系统中被移除，可以使用 TYPE_KEYGUARD_DIALOG 代替 |
| TYPE_TOAST | API 已经过时，用 TYPE_APPLICATION_OVERLAY 代替 |
| TYPE_SYSTEM_OVERLAY | API 已经过时，用 TYPE_APPLICATION_OVERLAY 代替 |
| TYPE_PRIORITY_PHONE | API 已经过时，用 TYPE_APPLICATION_OVERLAY 代替 |
| TYPE_SYSTEM_ERROR | API 已经过时，用 TYPE_APPLICATION_OVERLAY 代替 |
| TYPE_APPLICATION_OVERLAY | 应用程序叠加窗口显示在所有窗口之上 |
| TYPE_SYSTEM_DIALOG | 系统对话框窗口 |
| TYPE_KEYGUARD_DIALOG | 锁屏时显示的对话框 |
| TYPE_INPUT_METHOD | 输入法窗口，位于普通 UI 之上，应用程序可重新布局以免被此窗口覆盖 |
| TYPE_INPUT_METHOD_DIALOG | 输入法对话框，显示于当前输入法窗口之上 |
| TYPE_WALLPAPER | 墙纸 |
| TYPE_STATUS_BAR_PANEL | 状态栏的滑动面板 |
| LAST_SYSTEM_WINDOW | 系统 Window 类型的结束值 |


**需要注意的是：**

1. `TYPE_PHONE、TYPE_SYSTEM_ALERT、TYPE_TOAST、TYPE_SYSTEM_OVERLAY、TYPE_PRIORITY_PHONE、TYPE_SYSTEM_ERROR` 这些 type 在 API 26 中均已经过时，使用 `TYPE_APPLICATION_OVERLAY` 代替，**需要申请 Manifest.permission.SYSTEM_ALERT_WINDOW 权限**
    
2. `TYPE_KEYGUARD` 已经被从系统中移除，可以使用 `TYPE_KEYGUARD_DIALOG` 来代替

### Window 视图层级顺序

我们在手机上看的是二维的，但是实际上是三维的显示，如下图所示

![640](http://cdn.51git.cn/2020-06-03-640.gif)

在文章开头介绍了参数类型包含了 Window 的 x 轴坐标、Window 的 y 轴坐标， 既然是一个三维坐标系，那么 z 轴坐标在哪里？ 接下来就是我们要分析的非常重要的一个类 WindowManagerService，当添加 Window 的时候已经确定好了 Window 的层级，显示的时候才会根据当前的层级确定 Window 应该在哪一层显示

WindowManager 是 Window 的容器，管理着 Window，对 Window 进行添加和删除，具体的工作都是由 WMS 来处理的，WindowManager 和 WMS 通过 Binder 来进行跨进程通信，WMS 才是 Window 的最终管理者，我先来看一下 WMS 的 addWindow 方法
**frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java**

```
public int addWindow(Session session, IWindow client, int seq,
        LayoutParams attrs, int viewVisibility, int displayId, Rect outFrame,
        Rect outContentInsets, Rect outStableInsets, Rect outOutsets,
        DisplayCutout.ParcelableWrapper outDisplayCutout, InputChannel outInputChannel,
        InsetsState outInsetsState) {
        
            final WindowState win = new WindowState(this, session, client, token, parentWindow,
            appOp[0], seq, attrs, viewVisibility, session.mUid,
            session.mCanAddInternalSystemWindow);
            ......
            
            win.mToken.addWindow(win);
            ......
            
            win.getParent().assignChildLayers();
            ......
     
}
```

* WindowState 计算当前 Window 层级
* win.mToken.addWindow 这个方法将当前的 win 放入 WindowList 中，WindowList 是一个 ArrayList
* displayContent.assignWindowLayers 方法 计算 z-order 值, z-order 值越大越靠前，就越靠近用户

Window 视图层级顺序 用 Z-order 来表示，Z-order 对应着 WindowManager.LayoutParams 的 type 值，Z-order 可以理解为 Android 视图的层级概念，值越大越靠前，就越靠近用户。

WindowState 就是 windowManager 中的窗口，一个 WindowState 表示一个 window

那么 Z-order 的值的计算逻辑在 WindowState 类中，WindowState 构造的时候初始化当前的 mBaseLayer 和 mSubLayer，这两个参数应该是决定 z-order 的两个因素
**frameworks/base/services/core/java/com/android/server/wm/WindowState.java**

```
static final int TYPE_LAYER_MULTIPLIER = 10000;
static final int TYPE_LAYER_OFFSET = 1000;
    
WindowState(WindowManagerService service, Session s, IWindow c, WindowToken token,
        WindowState parentWindow, int appOp, int seq, WindowManager.LayoutParams a,
        int viewVisibility, int ownerId, boolean ownerCanAddInternalSystemWindow,
        PowerManagerWrapper powerManagerWrapper) {
      
        // 判断该是否在子 Window 的类型范围内[1000,1999]
        if (mAttrs.type >= FIRST_SUB_WINDOW && mAttrs.type <= LAST_SUB_WINDOW) {

            // 调用 getWindowLayerLw 方法返回值在[1,33]之间，根据不同类型的 Window 在屏幕上进行排序
            mBaseLayer = mPolicy.getWindowLayerLw(parentWindow)
                    * TYPE_LAYER_MULTIPLIER + TYPE_LAYER_OFFSET;

            // mSubLayer 子窗口的顺序
            // 调用 getSubWindowLayerFromTypeLw 方法返回值在[-2.3]之间 ，返回子 Window 相对于父 Window 的位置
            mSubLayer = mPolicy.getSubWindowLayerFromTypeLw(a.type);
            ......
            
        } else {
           
            mBaseLayer = mPolicy.getWindowLayerLw(this)
                    * TYPE_LAYER_MULTIPLIER + TYPE_LAYER_OFFSET;
            mSubLayer = 0;
            ......
            
        }  
}
```

* mBaseLayer 是基础序，对应的区间范围 [1,33]
* mSubLayer 相同分组下的子 Window 的序，对应的区间范围 [-2.3]
* 判断该是否在子 Window 的类型范围内[1000,1999]
* 如果是子 Window，调用 getWindowLayerLw 方法，计算 mBaseLayer 的值，返回一个用来对 Window 进行排序的任意整数，调用 getSubWindowLayerFromTypeLw 方法，计算 mSubLayer 的值，返回子 Window 相对于父 Window 的位置
* 如果不是子 Window，调用 getWindowLayerLw 方法，计算 mBaseLayer 的值，返回一个用来对 Window 进行排序的任意整数，mSubLayer 值为 0

#### 计算 mBaseLayer 的值

调用 WindowManagerPolicy 的 getWindowLayerLw 方法，计算 mBaseLayer 的值
**frameworks/base/services/core/java/com/android/server/policy/WindowManagerPolicy.java**

```
int APPLICATION_LAYER = 2;
int APPLICATION_MEDIA_SUBLAYER = -2;
int APPLICATION_MEDIA_OVERLAY_SUBLAYER = -1;
int APPLICATION_PANEL_SUBLAYER = 1;
int APPLICATION_SUB_PANEL_SUBLAYER = 2;
int APPLICATION_ABOVE_SUB_PANEL_SUBLAYER = 3;
 
/**
* 根据不同类型的 Window 在屏幕上进行排序
* 返回一个用来对窗口进行排序的任意整数，数字越小，表示的值越小
*/   
default int getWindowLayerFromTypeLw(int type, boolean canAddInternalSystemWindow) {
    // 判断是否在应用程序 Window 类型的取值范围内 [1,99]
    if (type >= FIRST_APPLICATION_WINDOW && type <= LAST_APPLICATION_WINDOW) {
        return APPLICATION_LAYER;
    }


    switch (type) {
        case TYPE_WALLPAPER: // 壁纸，通过 window manager 删除它
            return  1;
        case TYPE_PHONE: // 电话
            return  3;
        case TYPE_SEARCH_BAR: // 搜索栏
            return  6;
        case TYPE_SYSTEM_DIALOG: // 系统的 dialog
            return  7;
        case TYPE_TOAST: // 系统 toast
            return  8;
        case TYPE_INPUT_METHOD: // 输入法
            return  15;
        case TYPE_STATUS_BAR: // 状态栏
            return  17;
        case TYPE_KEYGUARD_DIALOG: //锁屏
            return  20;
        ......
        
        case TYPE_POINTER:
            // the (mouse) pointer layer
            return  33;
        default:
            return APPLICATION_LAYER;
    }
}
```

根据不同类型的 Window 在屏幕上进行排序，返回一个用来对 Window 进行排序的任意整数，数字越小，表示的值越小，通过以下公式来计算它的基础序 ，基础序越大，Z-order 值越大越靠前，就越靠近用户，我们以 **Activity** 为例：

Activity 属于应用层 Window，它的取值范围在 [1,99] 内，调用 getWindowLayerLw 方法返回 APPLICATION_LAYER，APPLICATION_LAYER 值为 2，通过下面方法进行计算 

```
static final int TYPE_LAYER_MULTIPLIER = 10000;
static final int TYPE_LAYER_OFFSET = 1000;

mBaseLayer = mPolicy.getWindowLayerLw(parentWindow)
                * TYPE_LAYER_MULTIPLIER + TYPE_LAYER_OFFSET;
```

那么最终 Activity 的 mBaseLayer 值是 21000

#### 计算 mSubLayer 的值

调用 getSubWindowLayerFromTypeLw 方法 ，传入  WindowManager.LayoutParams 的实例 a 的 type 值，计算 mSubLayer 的值
**frameworks/base/services/core/java/com/android/server/policy/WindowManagerPolicy.java**

```
int APPLICATION_LAYER = 2;
int APPLICATION_MEDIA_SUBLAYER = -2;
int APPLICATION_MEDIA_OVERLAY_SUBLAYER = -1;
int APPLICATION_PANEL_SUBLAYER = 1;
int APPLICATION_SUB_PANEL_SUBLAYER = 2;
int APPLICATION_ABOVE_SUB_PANEL_SUBLAYER = 3;

/**
* 计算 Window 相对于父 Window 的位置
* 返回 一个整数，正值在前面，表示在父 Window 上面，负值在后面，表示在父 Window 的下面
*/
default int getSubWindowLayerFromTypeLw(int type) {
    switch (type) {
        case TYPE_APPLICATION_PANEL: // 1000
        case TYPE_APPLICATION_ATTACHED_DIALOG: // 1003
            return APPLICATION_PANEL_SUBLAYER; // return 1
        case TYPE_APPLICATION_MEDIA:// 1001
            return APPLICATION_MEDIA_SUBLAYER;// return -2
        case TYPE_APPLICATION_MEDIA_OVERLAY:
            return APPLICATION_MEDIA_OVERLAY_SUBLAYER; // return -1
        case TYPE_APPLICATION_SUB_PANEL:// 1002
            return APPLICATION_SUB_PANEL_SUBLAYER;// return 2
        case TYPE_APPLICATION_ABOVE_SUB_PANEL:
            return APPLICATION_ABOVE_SUB_PANEL_SUBLAYER;// return 3
    }
    return 0;
}
```

计算子 Window 相对于父 Window 的位置，返回一个整数，正值表示在父 Window 上面，负值表示在父 Window 的下面

### Window 的 flag

Window 的 flag 用于控制 Window 的显示，它们的值也是定义在 WindowManager 的内部类 LayoutParams 中
**frameworks/base/core/java/android/view/WindowManager#LayoutParams.java**

```
// 当 Window 可见时允许锁屏
public static final int FLAG_ALLOW_LOCK_WHILE_SCREEN_ON     = 0x00000001;

// Window 后面的内容都变暗
public static final int FLAG_DIM_BEHIND        = 0x00000002;

@Deprecated
// API 已经过时，Window 后面的内容都变模糊
public static final int FLAG_BLUR_BEHIND        = 0x00000004;

// Window 不能获得输入焦点，即不接受任何按键或按钮事件，例如该 Window 上 有 EditView，点击 EditView 是 不会弹出软键盘的
// Window 范围外的事件依旧为原窗口处理；例如点击该窗口外的view，依然会有响应。另外只要设置了此Flag，都将会启用FLAG_NOT_TOUCH_MODAL
public static final int FLAG_NOT_FOCUSABLE      = 0x00000008;

// 设置了该 Flag,将 Window 之外的按键事件发送给后面的 Window 处理, 而自己只会处理 Window 区域内的触摸事件
// Window 之外的 view 也是可以响应 touch 事件。
public static final int FLAG_NOT_TOUCH_MODAL    = 0x00000020;

// 设置了该Flag，表示该 Window 将不会接受任何 touch 事件，例如点击该 Window 不会有响应，只会传给下面有聚焦的窗口。
public static final int FLAG_NOT_TOUCHABLE      = 0x00000010;

// 只要 Window 可见时屏幕就会一直亮着
public static final int FLAG_KEEP_SCREEN_ON     = 0x00000080;

// 允许 Window 占满整个屏幕
public static final int FLAG_LAYOUT_IN_SCREEN   = 0x00000100;

// 允许 Window 超过屏幕之外
public static final int FLAG_LAYOUT_NO_LIMITS   = 0x00000200;

// 全屏显示，隐藏所有的 Window 装饰，比如在游戏、播放器中的全屏显示
public static final int FLAG_FULLSCREEN      = 0x00000400;

// 表示比FLAG_FULLSCREEN低一级，会显示状态栏
public static final int FLAG_FORCE_NOT_FULLSCREEN   = 0x00000800;

// 当用户的脸贴近屏幕时（比如打电话），不会去响应此事件
public static final int FLAG_IGNORE_CHEEK_PRESSES    = 0x00008000;

// 则当按键动作发生在 Window 之外时，将接收到一个MotionEvent.ACTION_OUTSIDE事件。
public static final int FLAG_WATCH_OUTSIDE_TOUCH = 0x00040000;

@Deprecated
// 窗口可以在锁屏的 Window 之上显示, 使用 Activity#setShowWhenLocked(boolean) 方法代替
public static final int FLAG_SHOW_WHEN_LOCKED = 0x00080000;

// 表示负责绘制系统栏背景。如果设置，系统栏将以透明背景绘制，
// 此 Window 中的相应区域将填充 Window＃getStatusBarColor（）和 Window＃getNavigationBarColor（）中指定的颜色。
public static final int FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS = 0x80000000;

// 表示要求系统壁纸显示在该 Window 后面，Window 表面必须是半透明的，才能真正看到它背后的壁纸
public static final int FLAG_SHOW_WALLPAPER = 0x00100000;
```

| <div style="width: 170pt"> flag </div> | 备注 |
| --- | --- |
| FLAG_ALLOW_LOCK_WHILE_SCREEN_ON | 当 Window 可见时允许锁屏 |
| FLAG_DIM_BEHIND | Window 后面的内容都变暗 |
| FLAG_BLUR_BEHIND | API 已经过时，Window 后面的内容都变模糊 |
| FLAG_NOT_FOCUSABLE | Window 不能获得输入焦点，即不接受任何按键或按钮事件，例如该 Window 上 有 EditView，点击 EditView 是 不会弹出软键盘的，Window 范围外的事件依旧为原窗口处理；例如点击该窗口外的view，依然会有响应。另外只要设置了此Flag，都将会启用FLAG_NOT_TOUCH_MODAL |
| FLAG_NOT_TOUCH_MODAL | 设置了该 Flag,将 Window 之外的按键事件发送给后面的 Window 处理, 而自己只会处理 Window 区域内的触摸事件，Window 之外的 view 也是可以响应 touch 事件|
| FLAG_NOT_TOUCHABLE | 设置了该Flag，表示该 Window 将不会接受任何 touch 事件，例如点击该 Window 不会有响应，只会传给下面有聚焦的窗口 |
| FLAG_KEEP_SCREEN_ON | 只要 Window 可见时屏幕就会一直亮着 |
| FLAG_LAYOUT_IN_SCREEN | 允许 Window 占满整个屏幕 |
| FLAG_LAYOUT_NO_LIMITS | 允许 Window 超过屏幕之外 |
| FLAG_FULLSCREEN | 全屏显示，隐藏所有的 Window 装饰，比如在游戏、播放器中的全屏显示 |
| FLAG_FORCE_NOT_FULLSCREEN | 表示比FLAG_FULLSCREEN低一级，会显示状态栏 |
| FLAG_IGNORE_CHEEK_PRESSES | 当用户的脸贴近屏幕时（比如打电话），不会去响应此事件 |
| FLAG_WATCH_OUTSIDE_TOUCH | 则当按键动作发生在 Window 之外时，将接收到一个MotionEvent.ACTION_OUTSIDE事件 |
| FLAG_SHOW_WHEN_LOCKED | 已经过时，窗口可以在锁屏的 Window 之上显示, 使用 Activity#setShowWhenLocked(boolean) 方法代替 |
| FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS | 表示负责绘制系统栏背景。如果设置，系统栏将以透明背景绘制，<br/> 此 Window 中的相应区域将填充 Window＃getStatusBarColor（）和 Window＃getNavigationBarColor（）中指定的颜色|
| FLAG_SHOW_WALLPAPER | 表示要求系统壁纸显示在该 Window 后面，Window 表面必须是半透明的，才能真正看到它背后的壁纸 |

### window 软键盘模式

表示 window 软键盘输入区域的显示模式，常见的情况 Window 的软键盘打开会占据整个屏幕，遮挡了后面的视图，例如看直播的时候底部有个输入框点击的时候，输入框随着键盘一起上来，而有的时候，希望键盘覆盖在所有的 View 之上，界面保持不动等等

软键盘模式(SoftInputMode) 值，与 AndroidManifest 中 Activity 的属性 android:windowSoftInputMode 是对应的，因此可以在 AndroidManifest 文件中为 Activity 设置android:windowSoftInputMode

```
 <activity android:windowSoftInputMode="adjustNothing" />
```

也可以在 Java 代码中为 Window 设置 SoftInputMode

```
getWindow().setSoftInputMode(WindowManager.LayoutParams.SOFT_INPUT_ADJUST_NOTHING);
```

**SoftInputMode 常用的有以下几个值**

```
// 不会改变软键盘的状态
public static final int SOFT_INPUT_STATE_UNCHANGED = 1;

// 当用户进入该窗口时，隐藏软键盘
public static final int SOFT_INPUT_STATE_HIDDEN = 2;

// 当窗口获取焦点时，隐藏软键盘
public static final int SOFT_INPUT_STATE_ALWAYS_HIDDEN = 3;

// 当用户进入窗口时，显示软键盘
public static final int SOFT_INPUT_STATE_VISIBLE = 4;

// 当窗口获取焦点时，显示软键盘
public static final int SOFT_INPUT_STATE_ALWAYS_VISIBLE = 5;

// window会调整大小以适应软键盘窗口
public static final int SOFT_INPUT_MASK_ADJUST = 0xf0;

// 没有指定状态,系统会选择一个合适的状态或依赖于主题的设置
public static final int SOFT_INPUT_ADJUST_UNSPECIFIED = 0x00;

// 当软键盘弹出时，窗口会调整大小,例如点击一个EditView，整个layout都将平移可见且处于软件盘的上方
// 同样的该模式不能与SOFT_INPUT_ADJUST_PAN结合使用；
// 如果窗口的布局参数标志包含FLAG_FULLSCREEN，则将忽略这个值，窗口不会调整大小，但会保持全屏。
public static final int SOFT_INPUT_ADJUST_RESIZE = 0x10;

// 当软键盘弹出时，窗口不需要调整大小, 要确保输入焦点是可见的,
// 例如有两个EditView的输入框，一个为Ev1，一个为Ev2，当你点击Ev1想要输入数据时，当前的Ev1的输入框会移到软键盘上方
// 该模式不能与SOFT_INPUT_ADJUST_RESIZE结合使用
public static final int SOFT_INPUT_ADJUST_PAN = 0x20;

// 将不会调整大小，直接覆盖在window上
public static final int SOFT_INPUT_ADJUST_NOTHING = 0x30;
```

| <div style="width: 210pt"> model </div>  | 备注 |
| --- | --- |
| SOFT_INPUT_STATE_UNCHANGED | 不会改变软键盘的状态 |
| SOFT_INPUT_STATE_VISIBLE | 当用户进入窗口时，显示软键盘  |
| SOFT_INPUT_STATE_HIDDEN | 当用户进入该窗口时，隐藏软键盘 |
| SOFT_INPUT_STATE_ALWAYS_HIDDEN | 当窗口获取焦点时，隐藏软键盘 |
| SOFT_INPUT_STATE_ALWAYS_VISIBLE | 当窗口获取焦点时，显示软键盘 |
| SOFT_INPUT_MASK_ADJUST  | window 会调整大小以适应软键盘窗口  |
| SOFT_INPUT_ADJUST_UNSPECIFIED | 没有指定状态,系统会选择一个合适的状态或依赖于主题的设置 |
| SOFT_INPUT_ADJUST_RESIZE | 1. 当软键盘弹出时，窗口会调整大小,例如点击一个EditView，整个layout都将平移可见且处于软件盘的上方<br/> 2. 同样的该模式不能与SOFT_INPUT_ADJUST_PAN结合使用<br/> 3. 如果窗口的布局参数标志包含FLAG_FULLSCREEN，则将忽略这个值，窗口不会调整大小，但会保持全屏|
| SOFT_INPUT_ADJUST_PAN | 1. 当软键盘弹出时，窗口不需要调整大小, 要确保输入焦点是可见的<br/> 2. 例如有两个EditView的输入框，一个为Ev1，一个为Ev2，当你点击Ev1想要输入数据时，当前的Ev1的输入框会移到软键盘上方<br/> 3. 该模式不能与SOFT_INPUT_ADJUST_RESIZE结合使用|
| SOFT_INPUT_ADJUST_NOTHING | 将不会调整大小，直接覆盖在window上 |

## Kotlin 小技巧

利用 plus (+) 和 plus (-) 对 Map 集合做运算，如下所示：

```
fun main() {
    val numbersMap = mapOf("one" to 1, "two" to 2, "three" to 3)

    // plus (+)
    println(numbersMap + Pair("four", 4)) // {one=1, two=2, three=3, four=4}
    println(numbersMap + Pair("one", 10)) // {one=10, two=2, three=3}
    println(numbersMap + Pair("five", 5) + Pair("one", 11)) // {one=11, two=2, three=3, five=5}

    // plus (-)
    println(numbersMap - "one") // {two=2, three=3}
    println(numbersMap - listOf("two", "four")) // {one=1, three=3}
}
```

## 总结

到这里就结束了，这篇文章主要介绍了 Window 的类型大概可以分为三类： 应用程序 Window（Application Window）、子 Window（Sub Windwow）、系统 Window（System Window）。 分别介绍了 Window 的类型、Window 视图层级顺序、Window 的 flag、和 window 软键盘模式为后面的内容做铺垫

**Window 都有那些常用的参数?**

| 参数 | 备注 |
| --- | --- |
| x | window 左上角的 x 坐标 |
| y | window 左上角的 y 坐标 |
| type | Window 的类型 |
| flag | Window 的 flag 用于控制 Window 的显示 |
| softInputMode | window 软键盘输入区域的显示模式 |
| alpha | Window 的透明度，取值为0-1 |
| gravity | Window 在屏幕中的位置 |
| alpha | Window 的透明度，取值为0-1 |
| format | Window 的像素点格式，值定义在 PixelFormat 中 |

**Window 都有那些类型？**

应用程序 Window（Application Window）、子 Window（Sub Windwow）、系统 Window（System Window），子 Window 依附于父 Window 上，并且他们的坐标空间相对于父 Window 的，每个大类型又包含多个小类型，每个类型在上文的表格中已经列出来了，

**Window 那些过时的 API 以及处理方案？**

1. `TYPE_PHONE、TYPE_SYSTEM_ALERT、TYPE_TOAST、TYPE_SYSTEM_OVERLAY、TYPE_PRIORITY_PHONE、TYPE_SYSTEM_ERROR` 这些 type 在 API 26 中均已经过时，使用 `TYPE_APPLICATION_OVERLAY` 代替，需要申请 Manifest.permission.SYSTEM_ALERT_WINDOW 权限
    
2. `TYPE_KEYGUARD` 已经被从系统中移除，可以使用 `TYPE_KEYGUARD_DIALOG` 来代替

**Window 视图层级顺序是如何确定的？**

Window 的参数 x、y，分别表示 Window 左上角的 x 坐标，Window 左上角的 y 坐标，Window 视图层级顺序 用 Z-order 来表示，Z-order 对应着 WindowManager.LayoutParams 的 type 值，Z-order 可以理解为 Android 视图的层级概念，值越大越靠前，就越靠近用户。而 mBaseLayer 和 mSubLayer 决定 z-order 的两个因素

**Window 都那些 flag？**

Window 的 flag 用于控制 Window 的显示，flag 的参数如下所示：

| <div style="width: 170pt"> flag </div> | 备注 |
| --- | --- |
| FLAG_ALLOW_LOCK_WHILE_SCREEN_ON | 当 Window 可见时允许锁屏 |
| FLAG_DIM_BEHIND | Window 后面的内容都变暗 |
| FLAG_BLUR_BEHIND | API 已经过时，Window 后面的内容都变模糊 |
| FLAG_NOT_FOCUSABLE | Window 不能获得输入焦点，即不接受任何按键或按钮事件，例如该 Window 上 有 EditView，点击 EditView 是 不会弹出软键盘的，Window 范围外的事件依旧为原窗口处理；例如点击该窗口外的view，依然会有响应。另外只要设置了此Flag，都将会启用FLAG_NOT_TOUCH_MODAL |
| FLAG_NOT_TOUCH_MODAL | 设置了该 Flag,将 Window 之外的按键事件发送给后面的 Window 处理, 而自己只会处理 Window 区域内的触摸事件，Window 之外的 view 也是可以响应 touch 事件|
| FLAG_NOT_TOUCHABLE | 设置了该Flag，表示该 Window 将不会接受任何 touch 事件，例如点击该 Window 不会有响应，只会传给下面有聚焦的窗口 |
| FLAG_KEEP_SCREEN_ON | 只要 Window 可见时屏幕就会一直亮着 |
| FLAG_LAYOUT_IN_SCREEN | 允许 Window 占满整个屏幕 |
| FLAG_LAYOUT_NO_LIMITS | 允许 Window 超过屏幕之外 |
| FLAG_FULLSCREEN | 全屏显示，隐藏所有的 Window 装饰，比如在游戏、播放器中的全屏显示 |
| FLAG_FORCE_NOT_FULLSCREEN | 表示比FLAG_FULLSCREEN低一级，会显示状态栏 |
| FLAG_IGNORE_CHEEK_PRESSES | 当用户的脸贴近屏幕时（比如打电话），不会去响应此事件 |
| FLAG_WATCH_OUTSIDE_TOUCH | 则当按键动作发生在 Window 之外时，将接收到一个MotionEvent.ACTION_OUTSIDE事件 |
| FLAG_SHOW_WHEN_LOCKED | 已经过时，窗口可以在锁屏的 Window 之上显示, 使用 Activity#setShowWhenLocked(boolean) 方法代替 |
| FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS | 表示负责绘制系统栏背景。如果设置，系统栏将以透明背景绘制，<br/> 此 Window 中的相应区域将填充 Window＃getStatusBarColor（）和 Window＃getNavigationBarColor（）中指定的颜色|
| FLAG_SHOW_WALLPAPER | 表示要求系统壁纸显示在该 Window 后面，Window 表面必须是半透明的，才能真正看到它背后的壁纸 |

**Window 软键盘模式？**

Window 的软键盘模式表示 Window 软键盘输入区域的显示模式

| <div style="width: 210pt"> model </div>  | 备注 |
| --- | --- |
| SOFT_INPUT_STATE_UNCHANGED | 不会改变软键盘的状态 |
| SOFT_INPUT_STATE_VISIBLE | 当用户进入窗口时，显示软键盘  |
| SOFT_INPUT_STATE_HIDDEN | 当用户进入该窗口时，隐藏软键盘 |
| SOFT_INPUT_STATE_ALWAYS_HIDDEN | 当窗口获取焦点时，隐藏软键盘 |
| SOFT_INPUT_STATE_ALWAYS_VISIBLE | 当窗口获取焦点时，显示软键盘 |
| SOFT_INPUT_MASK_ADJUST  | window 会调整大小以适应软键盘窗口  |
| SOFT_INPUT_ADJUST_UNSPECIFIED | 没有指定状态,系统会选择一个合适的状态或依赖于主题的设置 |
| SOFT_INPUT_ADJUST_RESIZE | 1. 当软键盘弹出时，窗口会调整大小,例如点击一个EditView，整个layout都将平移可见且处于软件盘的上方<br/> 2. 同样的该模式不能与SOFT_INPUT_ADJUST_PAN结合使用<br/> 3. 如果窗口的布局参数标志包含FLAG_FULLSCREEN，则将忽略这个值，窗口不会调整大小，但会保持全屏|
| SOFT_INPUT_ADJUST_PAN | 1. 当软键盘弹出时，窗口不需要调整大小, 要确保输入焦点是可见的<br/> 2. 例如有两个EditView的输入框，一个为Ev1，一个为Ev2，当你点击Ev1想要输入数据时，当前的Ev1的输入框会移到软键盘上方<br/> 3. 该模式不能与SOFT_INPUT_ADJUST_RESIZE结合使用|
| SOFT_INPUT_ADJUST_NOTHING | 将不会调整大小，直接覆盖在window上 |

## 参考文献

* [https://developer.android.google.cn/.../WindowManager.LayoutParams](https://developer.android.google.cn/reference/android/view/WindowManager.LayoutParams.html?hl=en#summary)
* [https://juejin.im/post/5d20658d6fb9a07ecd3d7e08](https://juejin.im/post/5d20658d6fb9a07ecd3d7e08)
* [https://www.jianshu.com/p/3528255475a2](https://www.jianshu.com/p/3528255475a2)

## 结语

致力于分享一系列 Android 系统源码、逆向分析、算法、翻译、Jetpack  源码相关的文章，如果你同我一样喜欢研究 Android 源码，可以关注我，如果你喜欢这篇文章欢迎 star，一起来学习，期待与你一起成长

**算法**

由于 LeetCode 的题库庞大，每个分类都能筛选出数百道题，由于每个人的精力有限，不可能刷完所有题目，因此我按照经典类型题目去分类、和题目的难易程度去排序

* 数据结构： 数组、栈、队列、字符串、链表、树……
* 算法： 查找算法、搜索算法、位运算、排序、数学、……

每道题目都会用 Java 和 kotlin 去实现，并且每道题目都有解题思路，如果你同我一样喜欢算法、LeetCode，可以关注我 GitHub 上的 LeetCode 题解：[Leetcode-Solutions-with-Java-And-Kotlin](https://github.com/hi-dhl/Leetcode-Solutions-with-Java-And-Kotlin)，一起来学习，期待与你一起成长


