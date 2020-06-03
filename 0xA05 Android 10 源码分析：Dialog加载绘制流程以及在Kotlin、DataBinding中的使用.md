# 0xA05 Android 10 源码分析：Dialog加载绘制流程以及在Kotlin、DataBinding中的使用

![DataBindingDialog1](http://cdn.51git.cn/2020-04-12-DataBindingDialog1.png)

## 引言

> * 这是 Android 10 源码分析系列的第 5 篇
> * 分支：android-10.0.0_r14
> * 全文阅读大概 10 分钟

**通过这篇文章你将学习到以下内容，将在文末总结部分会给出相应的答案**

* Dialog的的创建流程？
* Dialog的视图怎么与Window做关联了？
* 自定义CustomDialog的view的是如何绑定的?
* 如何使用Kotlin具名可选参数构造类，实现构建者模式？
* 相比于Java的构建者模式，通过具名可选参数构造类具有以下优点?
* 如何在Dialog中使用DataBinding？

阅读本文之前，如果之前没有看过 **Apk加载流程之资源加载一** 和 **Apk加载流程之资源加载二** 点击下方链接前去查看，这几篇文章都是互相关联的

* [0xA03 Android 10 源码分析：Apk加载流程之资源加载（一）](https://juejin.im/post/5e6c8c14f265da574b792a1a)
* [0xA04 Android 10 源码分析：Apk加载流程之资源加载（二）](https://juejin.im/post/5e7f0f2c51882573c4676bc7)

**本文主要来主要围绕以下几个方面来分析:**

* Dialog加载绘制流程
* 如何使用Kotlin具名可选参数构造类，实现构建者模式
* 如何在Dialog中使用DataBinding

## 源码分析

在开始分析Dialog的源码之前，需要了解一下Dialog加载绘制流程，涉及到的数据结构与职能

**在包 android.app 下：**

* Dialog：Dialog是窗口的父类，主要实现Window对象的初始化和一些共有逻辑
    * AlertDialog：继承自Dialog，是具体的Dialog的操作实现类
    * AlertDialog.Builder：是AlertDialog的内部类，主要用于构造AlertDialog
* AlertController：是AlertDialog的控制类
    * AlertController.AlertParams：是AlertController的内部类，负责AlertDialog的初始化参数

了解完相关的数据结构与职能，接下来回顾一下Dialog的创建流程

```
AlertDialog.Builder builder = new AlertDialog.Builder(this);
builder.setIcon(R.mipmap.ic_launcher);
builder.setMessage("Message部分");
builder.setTitle("Title部分");
builder.setView(R.layout.activity_main);

builder.setPositiveButton("确定", new DialogInterface.OnClickListener() {
    @Override
    public void onClick(DialogInterface dialog, int which) {
        alertDialog.dismiss();
    }
});
builder.setNegativeButton("取消", new DialogInterface.OnClickListener() {
    @Override
    public void onClick(DialogInterface dialog, int which) {
        alertDialog.dismiss();
    }
});
alertDialog = builder.create();
alertDialog.show();
```

上面代码都不会很陌生，主要使用了设计模式当中-构建者模式，

1. 构建AlertDialog.Builder对象
2. builder.setXXX 系列方法完成Dialog的初始化
3. 调用builder.create()方法创建AlertDialog
4. 调用AlertDialog的show()完成View的绘制并显示AlertDialog

主要通过上面四步完成Dialog的创建和显示，接下来根据源码来分析每个方法的具体实现，以及Dialog的视图怎么与Window做关联

### 1 构建AlertDialog.Builder对象

```
AlertDialog.Builder builder = new AlertDialog.Builder(this);
```

AlertDialog.Builder是AlertDialog的内部类，用于封装AlertDialog的构造过程，看一下Builder的构造方法
**frameworks/base/core/java/android/app/AlertDialog.java**

```
// AlertController.AlertParams类型的成员变量
private final AlertController.AlertParams P;
public Builder(Context context) {
    this(context, resolveDialogTheme(context, Resources.ID_NULL));
}
public Builder(Context context, int themeResId) {
    // 构造ContextThemeWrapper，ContextThemeWrapper 是 Context的子类，主要用来处理和主题相关的
    // 初始化成为变量 P
    P = new AlertController.AlertParams(new ContextThemeWrapper(
            context, resolveDialogTheme(context, themeResId)));
}
```

* ContextThemeWrapper 继承自ContextWrapper，Application、Service继承自ContextWrapper，Activity继承自ContextThemeWrapper

![context3](http://cdn.51git.cn/2020-04-06-context3.png)

* P是AlertDialog.Builder中的AlertController.AlertParams类型的成员变量
* AlertParams中包含了与AlertDialog视图中对应的成员变量，调用builder.setXXX系列方法之后，我们传递的参数就保存在P中了

### 1.1 AlertParams封装了初始化参数

AlertController.AlertParams 是AlertController的内部类，负责AlertDialog的初始化参数
**frameworks/base/core/java/com/android/internal/app/AlertController.java**

```
public AlertParams(Context context) {
mContext = context;
// mCancelable 用来控制点击外部是否可取消，默认可以取消
mCancelable = true;
// LayoutInflater 主要来解析layout.xml文件
mInflater = (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
}
```

* 主要执行了AlertController.AlertParams的初始化操作，初始化了一些成员变量，LayoutInflater 主要来解析layout.xml文件，关于LayoutInflater可以参考之前的文章[0xA04 Android 10 源码分析：Apk加载流程之资源加载（二）](https://juejin.im/post/5e7f0f2c51882573c4676bc7)
* 初始化完成AlertParams之后，就完成了AlertDialog.Builder的构建

### 2 调用AlertDialog.Builder的setXXX系列方法

AlertDialog.Builder初始化完成之后，调用它的builder.setXXX 系列方法完成Dialog的初始化
**frameworks/base/core/java/android/app/AlertDialog.java**

```
// ... 省略了很多builder.setXXX方法
public Builder setTitle(@StringRes int titleId) {
    P.mTitle = P.mContext.getText(titleId);
    return this;
}
public Builder setMessage(@StringRes int messageId) {
    P.mMessage = P.mContext.getText(messageId);
    return this;
}
public Builder setPositiveButton(@StringRes int textId, final OnClickListener listener) {
    P.mPositiveButtonText = P.mContext.getText(textId);
    P.mPositiveButtonListener = listener;
    return this;
}
public Builder setPositiveButton(CharSequence text, final OnClickListener listener) {
    P.mPositiveButtonText = text;
    P.mPositiveButtonListener = listener;
    return this;
}
// ... 省略了很多builder.setXXX方法
```

上面所有setXXX方法都是给Builder的成员变量P赋值，并且他们的返回值都是Builder类型，因此可以通过消息琏的方式调用

```
builder.setTitle().setMessage().setPositiveButton()...
```

PS: 在Kotlin应该尽量避免使用构建者模式，使用Kotlin中的具名可选参数，实现构建者模式，代码更加简洁，为了不影响阅读的流畅性，将这部分内容放到了文末**扩展阅读**部分

### 3 builder.create方法

builder.setXXX 系列方法之后调用builder.create方法完成AlertDialog构建，接下来看一下create方法
**frameworks/base/core/java/android/app/AlertDialog.java**

```
public AlertDialog create() {
    // P.mContext 是ContextWrappedTheme 的实例
    final AlertDialog dialog = new AlertDialog(P.mContext, 0, false);
    // Dialog的参数其实保存在P这个类里面
    // mAler是AlertController的实例，通过这个方法把P中的变量传给AlertController.AlertParams
    P.apply(dialog.mAlert);
    // 用来控制点击外部是否可取消,mCancelable 默认为true
    dialog.setCancelable(P.mCancelable);
    // 如果可以取消设置回调监听
    if (P.mCancelable) {
        dialog.setCanceledOnTouchOutside(true);
    }
    // 设置一系列监听
    dialog.setOnCancelListener(P.mOnCancelListener);
    dialog.setOnDismissListener(P.mOnDismissListener);
    if (P.mOnKeyListener != null) {
        dialog.setOnKeyListener(P.mOnKeyListener);
    }
    // 返回 AlertDialog 对象
    return dialog;
}
```

* 根据P.mContex 构建了一个AlertDialog
* mAler是AlertController的实例，调用apply方法把P中的变量传给AlertController.AlertParams
* 设置是否可以点击外部取消，默认可以取消，同时设置回调监听
* 最后返回AlertDialog对象

#### 3.1 如何构建AlertDialog

我们来分析一下AlertDialog是如何构建的，来看一下它的造方法具体实现
**frameworks/base/core/java/android/app/AlertDialog.java**

```
AlertDialog(Context context, @StyleRes int themeResId, boolean createContextThemeWrapper) {
    super(context, createContextThemeWrapper ? resolveDialogTheme(context, themeResId) : 0,
            createContextThemeWrapper);

    mWindow.alwaysReadCloseOnTouchAttr();
    // getContext() 返回的是ContextWrapperTheme
    // getWindow() 返回的是 PhoneWindow
    // mAlert 是AlertController的实例
    mAlert = AlertController.create(getContext(), this, getWindow());
}
```

PhoneWindows是什么时候创建的？AlertDialog继承自Dialog，首先调用了super的构造方法，来看一下Dialog的构造方法
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
    w.setCallback(this);
    w.setOnWindowDismissedCallback(this);
    w.setOnWindowSwipeDismissedCallback(() -> {
        if (mCancelable) {
            cancel();
        }
    });
    w.setWindowManager(mWindowManager, null, null);
    w.setGravity(Gravity.CENTER);
    // 继承 Handler
    mListenersHandler = new ListenersHandler(this);
}
```

* 获取WindowManager对象，构建了PhoneWindow，到这里我们知道了PhoneWindow是在Dialog构造方法创建的
* 初始化了Dialog的成员变量mWindow，mWindow 是PhoneWindow的实例
* 初始化了Dialog的成员变量mListenersHandler，mListenersHandler继承Handler

我们回到AlertDialog构造方法，在AlertDialog构造方法内，调用了 AlertController.create方法，来看一下这个方法

```
public static final AlertController create(Context context, DialogInterface di, Window window) {
    final TypedArray a = context.obtainStyledAttributes(
            null, R.styleable.AlertDialog, R.attr.alertDialogStyle,
            R.style.Theme_DeviceDefault_Settings);
    int controllerType = a.getInt(R.styleable.AlertDialog_controllerType, 0);
    a.recycle();
    // 根据controllerType 使用不同的AlertController
    switch (controllerType) {
        case MICRO:
            // MicroAlertController 是matrix风格 继承自AlertController
            return new MicroAlertController(context, di, window);
        default:
            return new AlertController(context, di, window);
    }
}
```

根据controllerType 返回不同的AlertController，到这里分析完了AlertDialog是如何构建的

### 4 调用Dialog的show方法显示Dialog

调用AlertDialog.Builder的create方法之后返回了AlertDialog的实例，最后调用了AlertDialog的show方法显示dialog，但是AlertDialog是继承自Dialog的，实际上调用的是Dialog的show方法
**frameworks/base/core/java/android/app/Dialog.java**

```
public void show() {
    // mShowing变量用于表示当前dialog是否正在显示
    if (mShowing) {
        if (mDecor != null) {
            if (mWindow.hasFeature(Window.FEATURE_ACTION_BAR)) {
                mWindow.invalidatePanelMenu(Window.FEATURE_ACTION_BAR);
            }
            mDecor.setVisibility(View.VISIBLE);
        }
        return;
    }
    
    mCanceled = false;

    // mCreated这个变量控制dispatchOnCreate方法只被执行一次
    if (!mCreated) {
        dispatchOnCreate(null);
    } else {
        // Fill the DecorView in on any configuration changes that
        // may have occured while it was removed from the WindowManager.
        final Configuration config = mContext.getResources().getConfiguration();
        mWindow.getDecorView().dispatchConfigurationChanged(config);
    }

    // 用于设置ActionBar
    onStart();
    // 获取DecorView
    mDecor = mWindow.getDecorView();

    if (mActionBar == null && mWindow.hasFeature(Window.FEATURE_ACTION_BAR)) {
        final ApplicationInfo info = mContext.getApplicationInfo();
        mWindow.setDefaultIcon(info.icon);
        mWindow.setDefaultLogo(info.logo);
        mActionBar = new WindowDecorActionBar(this);
    }
    // 获取布局参数
    WindowManager.LayoutParams l = mWindow.getAttributes();
    boolean restoreSoftInputMode = false;
    if ((l.softInputMode
            & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION) == 0) {
        l.softInputMode |=
                WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION;
        restoreSoftInputMode = true;
    }
    // 将DecorView和布局参数添加到WindowManager中，完成view的绘制
    mWindowManager.addView(mDecor, l);
    if (restoreSoftInputMode) {
        l.softInputMode &=
                ~WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION;
    }

    mShowing = true;
    // 向Handler发送一个Dialog的消息，从而显示AlertDialog
    sendShowMessage();
}
```

* 判断dialog是否已经显示，如果显示了直接返回
* 判断dispatchOnCreate方法是否已经调用，如果没有则调用dispatchOnCreate方法
* 获取布局参数添加到WindowManager，调用addView方法完成view的绘制
* 向Handler发送一个Dialog的消息，从而显示AlertDialog

#### 4.1 dispatchOnCreate

在上面代码中，根据mCreated变量，判断dispatchOnCreate方法是否已经调用，如果没有则调用dispatchOnCreate方法
**frameworks/base/core/java/android/app/Dialog.java**

```
void dispatchOnCreate(Bundle savedInstanceState) {
    if (!mCreated) {
        // 调用 onCreate 方法
        onCreate(savedInstanceState);
        mCreated = true;
    }
}
```

在dispatchOnCreate方法中主要调用Dialog的onCreate方法, Dialog的onCreate方法是个空方法，由于我们创建的是AlertDialog对象，AlertDialog继承于Dialog，所以调用的是AlertDialog的onCreate方法
**frameworks/base/core/java/android/app/AlertDialog.java**

```
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    mAlert.installContent();
}
```

在这方法里面调用了AlertController的installContent方法，来看一下具体的实现逻辑
**frameworks/base/core/java/com/android/internal/app/AlertController.java**

```
public void installContent() {
    // 获取相应的Dialog布局文件
    int contentView = selectContentView();
    // 调用setContentView方法解析布局文件
    mWindow.setContentView(contentView);
    // 初始化布局文件中的组件
    setupView();
}
```

* 调用selectContentView方法获取布局文件，来看一下具体的实现
**frameworks/base/core/java/com/android/internal/app/AlertController.java**

```
private int selectContentView() {
    if (mButtonPanelSideLayout == 0) {
        return mAlertDialogLayout;
    }
    if (mButtonPanelLayoutHint == AlertDialog.LAYOUT_HINT_SIDE) {
        return mButtonPanelSideLayout;
    }
    return mAlertDialogLayout;
}
```

返回的布局是mAlertDialogLayout，布局文件是在AlertController的构造方法初始化的
**frameworks/base/core/java/com/android/internal/app/AlertController.java**

```
mAlertDialogLayout = a.getResourceId(
                R.styleable.AlertDialog_layout, R.layout.alert_dialog);
```

* 调用Window.setContentView方法解析布局文件，Activity的setContentView最后也是调用了Window.setContentView这个方法，具体的解析流程，可以参考之前的文章Activity布局加载流程 [0xA03 Android 10 源码分析：Apk加载流程之资源加载](https://juejin.im/post/5e6c8c14f265da574b792a1a)
* 调用setupView方法初始化布局文件中的组件, 到这里dispatchOnCreate方法分析结束

#### 4.2 调用mWindowManager.addView完成View的绘制

回到我们的Dialog的show方法，在执行了dispatchOnCreate方法之后，又调用了onStart方法，这个方法主要用于设置ActionBar，然后初始化WindowManager.LayoutParams对象，最后调用mWindowManager.addView()方法完成界面的绘制，绘制完成之后调用sendShowMessage方法
**frameworks/base/core/java/android/app/Dialog.java**

```
private void sendShowMessage() {
    if (mShowMessage != null) {
        // Obtain a new message so this dialog can be re-used
        Message.obtain(mShowMessage).sendToTarget();
    }
}
```

向Handler发送一个Dialog的消息，从而显示AlertDialog，该消息最终会在ListenersHandler中的handleMessage方法中被执行，ListenersHandler是Dialog的内部类，继承Handler
**frameworks/base/core/java/android/app/Dialog.java**

```
public void handleMessage(Message msg) {
    switch (msg.what) {
        case DISMISS:
            ((OnDismissListener) msg.obj).onDismiss(mDialog.get());
            break;
        case CANCEL:
            ((OnCancelListener) msg.obj).onCancel(mDialog.get());
            break;
        case SHOW:
            ((OnShowListener) msg.obj).onShow(mDialog.get());
            break;
    }
}
```

如果msg.what = SHOW，会执行OnShowListener.onShow方法，msg.what的值和OnShowListener调用setOnShowListener方法赋值的
**frameworks/base/core/java/android/app/Dialog.java**

```
public void setOnShowListener(@Nullable OnShowListener listener) {
    if (listener != null) {
        mShowMessage = mListenersHandler.obtainMessage(SHOW, listener);
    } else {
        mShowMessage = null;
    }
}
```

mListenersHandler构造了Message对象，当我们在Dialog中发送showMessage的时候，被mListenersHandler所接收

#### 4.3 自定义Dialog的view的是如何绑定的

在上文分析中根据mCreated变量，判断dispatchOnCreate方法是否已经调用，如果没有则调用dispatchOnCreate方法，在dispatchOnCreate方法中主要调用Dialog的onCreate方法，由于创建的是AlertDialog对象，AlertDialog继承于Dialog，所以实际调用的是AlertDialog的onCreate方法，来完成布局文件的解析，和布局文件中控件的初始化<br/>

同理我们自定义CustomDialog继承自Dialog，所以调用的是自定义CustomDialog的onCreate方法，代码如下

```
public class CustomDialog extends Dialog {
    Context mContext;
    // ... 省略构造方法
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        LayoutInflater inflater = (LayoutInflater) mContext
                .getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        View view = inflater.inflate(R.layout.custom_dialog, null);
        setContentView(view);
    }

}
```

在onCreate方法中调用了 Dialog的setContentView 方法, 来分析setContentView方法
**frameworks/base/core/java/android/app/Dialog.java**

```
public void setContentView(@NonNull View view) {
    mWindow.setContentView(view);
}
```

mWindow是PhoneWindow的实例，最后调用的是PhoneWindow的setContentView解析布局文件，Activity的setContentView最后也是调用了PhoneWindow的setContentView方法，具体的解析流程，可以参考之前的文章Activity布局加载流程 [0xA03 Android 10 源码分析：Apk加载流程之资源加载](https://juejin.im/post/5e6c8c14f265da574b792a1a)

## 总结

Dialog和Activity的显示逻辑是相似的都是内部管理这一个Window对象，用WIndow对象实现界面的加载与显示逻辑<br/>

**Dialog的的创建流程？**

1. 构建AlertDialog.Builder对象
2. builder.setXXX 系列方法完成Dialog的初始化
3. 调用builder.create()方法创建AlertDialog
4. 调用AlertDialog的show()初始化Dialog的布局文件，Window对象等，然后执行mWindowManager.addView方法，开始执行绘制View的操作，最终将Dialog显示出来

**Dialog的视图怎么与Window做关联了？**

* 在Dialog的构造方法中初始化了Window对象

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

* 调用Dialog的show方法，完成view的绘制和Dialog的显示

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

最终会通过WindowManager将DecorView添加到Window之中，用WIndow对象实现界面的加载与显示逻辑

**自定义CustomDialog的view的是如何绑定的?**

* 调用Dialog的show方法，在该方法内部会根据mCreated变量，判断dispatchOnCreate方法是否已经调用，如果没有则调用dispatchOnCreate方法，在dispatchOnCreate方法中主要调用Dialog的onCreate方法
* 自定义CustomDialog继承自Dialog，所以调用的是自定义CustomDialog的onCreate方法，在CustomDialog的onCreate方法中调用setContentView方法，最后调用的是PhoneWindow的setContentView解析布局文件，解析流程参考[0xA03 Android 10 源码分析：Apk加载流程之资源加载](https://juejin.im/post/5e6c8c14f265da574b792a1a)

**如何使用Kotlin具名可选参数构造类，实现构建者模式？**

这部分内容参考**扩展阅读**部分

**相比于Java的构建者模式，通过具名可选参数构造类具有以下优点?**

* 代码非常的简洁
* 每个参数名都可以显示的，声明对象时无须按照顺序书写，非常的灵活
* 构造函数中每个参数都是val声明的，在多线程并发业务场景中更加的安全
* Kotlin的require方法，让我们在参数约束上更加的友好

**如何在Dialog中使用DataBinding？**

这部分内容参考**扩展阅读**部分

## 扩展阅读

### 1. Kotlin实现构建者模式

刚才在上文中提到了，在Kotlin中应该尽量避免使用构建者模式，使用Kotlin的具名可选参数构造类，实现构建者模式，代码更加简洁<br/>

> 在 "Effective Java" 书中介绍构建者模式时，是这样子描述它的：本质上builder模式模拟了具名的可算参数，就像Ada和Python中的一样

关于Java用构建者模式实现自定义dialog，可以参考这边文章 [Builder Pattern in Java](http://codethataint.com/blog/builder-pattern/)，代码显得很长........幸运的是，Kotlin是一门拥有具名可选参数的变成语言，Kotlin中的函数和构造器都支持这一特性，接下里我们使用具名可选参数构造类，实现构建者模式，点击[JDataBinding](https://github.com/hi-dhl/JDataBinding)前往查看，核心代码如下：

```
class AppDialog(
    context: Context,
    val title: String? = null,
    val message: String? = null,
    val yes: AppDialog.() -> Unit
) : DataBindingDialog(context, R.style.AppDialog) {

    init {
        requireNotNull(message) { "message must be not null" }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        requestWindowFeature(Window.FEATURE_NO_TITLE)

        setContentView(root)
        display.text = message
        btnNo.setOnClickListener { dismiss() }
        btnYes.setOnClickListener { yes() }
    }
}
```

调用方式也更加的简单

```
AppDialog(
        context = this@MainActivity,
        message = msg,
        yes = {
            // do something
        }).show()
```

**相比于Java的构建者模式，通过具名可选参数构造类具有以下优点:**

* 代码非常的简洁
* 每个参数名都可以显示的，声明对象时无须按照顺序书写，非常的灵活
* 构造函数中每个参数都是val声明的，在多线程并发业务场景中更加的安全
* Kotlin的require方法，让我们在参数约束上更加的友好

### 2. 如何在Dialog中使用DataBinding

DataBinding是什么？查看[Google官网](https://developer.android.com/topic/libraries/data-binding)，会有更详细的介绍<br/>

> DataBinding 是 Google 在 Jetpack 中推出的一款数据绑定的支持库，利用该库可以实现在页面组件中直接绑定应用程序的数据源

在使用Kotlin的具名可选参数构造类实现Dailog构建者模式的基础上，用DataBinding进行二次封装，加上DataBinding数据绑定的特性，使Dialog变得更加简洁、易用<br/>

**Step1: 定义一个基类DataBindingDialog**

```
abstract class DataBindingDialog(@NonNull context: Context, @StyleRes themeResId: Int) :
    Dialog(context, themeResId) {

    protected inline fun <reified T : ViewDataBinding> binding(@LayoutRes resId: Int): Lazy<T> =
        lazy {
            requireNotNull(
                DataBindingUtil.bind<T>(LayoutInflater.from(context).inflate(resId, null))
            ) { "cannot find the matched view to layout." }
        }

}
```

**Step2:  改造AppDialog**

```
class AppDialog(
    context: Context,
    val title: String? = null,
    val message: String? = null,
    val yes: AppDialog.() -> Unit
) : DataBindingDialog(context, R.style.AppDialog) {
    private val mBinding: DialogAppBinding by binding(R.layout.dialog_app)

    init {
        requireNotNull(message) { "message must be not null" }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        requestWindowFeature(Window.FEATURE_NO_TITLE)

        mBinding.apply {
            setContentView(root)
            display.text = message
            btnNo.setOnClickListener { dismiss() }
            btnYes.setOnClickListener { yes() }
        }

    }
}
```

同理DataBinding在Activity、Fragment、Adapter中的使用也是一样的，利用Kotlin的inline、reified、DSL等等语法，可以设计出更加简洁并利于维护的代码<br/>

关于基于DataBinding封装的DataBindingActivity、DataBindingFragment、DataBindingDialog基础库相关代码，后续也会陆续完善基础库，点击[JDataBinding](https://github.com/hi-dhl/JDataBinding)前往查看，欢迎start<br/>

## 结语

致力于分享一系列 Android 系统源码、逆向分析、算法、翻译、Jetpack  源码相关的文章，如果你同我一样喜欢研究 Android 源码，可以关注我，如果你喜欢这篇文章欢迎 star，一起来学习，期待与你一起成长

**算法**

由于 LeetCode 的题库庞大，每个分类都能筛选出数百道题，由于每个人的精力有限，不可能刷完所有题目，因此我按照经典类型题目去分类、和题目的难易程度去排序

* 数据结构： 数组、栈、队列、字符串、链表、树……
* 算法： 查找算法、搜索算法、位运算、排序、数学、……

每道题目都会用 Java 和 kotlin 去实现，并且每道题目都有解题思路，如果你同我一样喜欢算法、LeetCode，可以关注我 GitHub 上的 LeetCode 题解：[Leetcode-Solutions-with-Java-And-Kotlin](https://github.com/hi-dhl/Leetcode-Solutions-with-Java-And-Kotlin)，一起来学习，期待与你一起成长

