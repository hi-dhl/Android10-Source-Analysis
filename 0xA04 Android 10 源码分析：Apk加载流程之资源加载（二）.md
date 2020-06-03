# 0xA04 Android 10 源码分析：Apk加载流程之资源加载（二）

![1_enjGzlIFq8vgKB0la0EoUg](http://cdn.51git.cn/2020-06-03-1_enjGzlIFq8vgKB0la0EoUg.jpeg)

## 引言

> * 这是 Android 10 源码分析系列的第 4 篇
> * 分支：android-10.0.0_r14
> * 全文阅读大概 10 分钟

**通过这篇文章你将学习到以下内容，将在文末会给出相应的答案**

* View中的INVISIBLE、VISIBLE、GONE都有什么作用？
* 为什么ViewStub是大小为0的视图
* ViewStub有什么作用？
* ViewStub是如何创建的？
* 为什么ViewStub能做到延迟加载？
* ViewStub指定的Layout布局文件是什么时候被加载的？
* LayoutInflater是一个抽象类它如何被创建的？
* 系统服务存储在哪里？如何获取和添加系统服务？

在上一篇文章 [0xA02 Android 10 源码分析：Apk加载流程之资源加载](https://juejin.im/post/5e6c8c14f265da574b792a1a) 中通过LayoutInflater.inflate方法解析xml文件，了解到了系统如何对merge、include标签是如何处理的，本文主要围绕以下两方面内容

* 系统对ViewStub如何处理？
* LayoutInflater是如何被创建的？

**系统对merge、include是如何处理的**

* 使用merge标签必须有父布局，且依赖于父布局加载
* merge并不是一个ViewGroup，也不是一个View，它相当于声明了一些视图，等待被添加，解析过程中遇到merge标签会将merge标签下面的所有子view添加到根布局中
* merge标签在 XML 中必须是根元素
* 相反的include不能作为根元素，需要放在一个ViewGroup中
* 使用 include 标签必须指定有效的 layout 属性
* 使用 include 标签不写宽高是没有关系的，会去解析被 include 的 layout

**merge标签为什么可以起到优化布局的效果？**

解析过程中遇到merge标签，会调用rInflate方法，部分代码如下

```
// 根据元素名解析，生成对应的view
final View view = createViewFromTag(parent, name, context, attrs);
final ViewGroup viewGroup = (ViewGroup) parent;
final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
// rInflateChildren方法内部调用的rInflate方法，深度优先遍历解析所有的子view
rInflateChildren(parser, view, attrs, true);
// 添加解析的view
viewGroup.addView(view, params);
```

解析merge标签下面的所有子view，然后添加到根布局中<br/>
更多信息查看[0xA02 Android 10 源码分析：Apk加载流程之资源加载](https://juejin.im/post/5e6c8c14f265da574b792a1a)，接下来看一下系统对ViewStub如何处理

## 1. ViewStub是什么

**关于ViewStub的介绍，可以点击下方官网链接查看**
[官网链接https://developer.android.google.cn/reference/android...ViewStub](https://developer.android.google.cn/reference/android/view/ViewStub/)

**ViewStub的继承结构**
![](http://cdn.51git.cn/2020-03-28-15848875296387.jpg)

![](http://cdn.51git.cn/2020-03-28-15848860166670.jpg)

**简单来说主要以下几点：**

* ViewStub控件是一个不可见，
* 大小为0的视图
* 当ViewStub控件设置可见，或者调用inflate()方法，ViewStub所指定的layout资源就会被加载
* ViewStub也会从其父控件中移除，ViewStub会被新加载的layout文件代替

**为什么ViewStub是大小为0的视图**
**frameworks/base/core/java/android/view/ViewStub.java**

```
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    // 设置视图大小为0
    setMeasuredDimension(0, 0);
}
```


**ViewStub的作用**

主要用来延迟布局的加载，例如：在Android中非常常见的布局，用ListView来展示列表信息，当没有数据或者网络加载失败时, 加载空的 ListView 会占用一些资源，如果用ViewStub包裹ListView，当有数据时，才会调用inflate()方法显示ListView，起到延迟加载了布局效果

### 1.1 ViewStub 是如何被创建的

在上篇文章[0xA02 Android 10 源码分析：Apk加载流程之资源加载](https://juejin.im/post/5e6c8c14f265da574b792a1a)中，介绍了view的创建是通过调用了LayoutInflater.createView方法根据完整的类的路径名利用反射机制构建View对象，因为ViewStub是继承View，所以ViewStub的创建和View的创建是相同的，来看一下LayoutInflater.createView方法
**frameworks/base/core/java/android/view/LayoutInflater.java**

```
...

try {
    // 利用构造函数，创建View
    final View view = constructor.newInstance(args);
    if (view instanceof ViewStub) {
        // 如果是ViewStub，则设置LayoutInflater
        final ViewStub viewStub = (ViewStub) view;
        viewStub.setLayoutInflater(cloneInContext((Context) args[0]));
    }
    return view;
} finally {
    mConstructorArgs[0] = lastContext;
}

...
```

根据完整的类的路径名利用反射机制构建View对象，如果遇到ViewStub将当前LayoutInflater设置给ViewStub，当ViewStub控件设置可见，或者调用inflate()，会调用LayoutInflater的inflate方法完成布局加载，接下来分析ViewStub的构造方法

### 1.2 ViewStub的构造方法

在上面提到了根据完整的类的路径名利用反射机制构建View对象，当view对象被创建的时候，会调用它的构造函数，来看一下ViewStub的构造方法

```
public ViewStub(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
    super(context);

    final TypedArray a = context.obtainStyledAttributes(attrs,
            R.styleable.ViewStub, defStyleAttr, defStyleRes);
    saveAttributeDataForStyleable(context, R.styleable.ViewStub, attrs, a, defStyleAttr,
            defStyleRes);
    // 解析xml中设置的 android:inflatedId 的属性
    mInflatedId = a.getResourceId(R.styleable.ViewStub_inflatedId, NO_ID);
    // 解析xml中设置的 android:layout 属性
    mLayoutResource = a.getResourceId(R.styleable.ViewStub_layout, 0);
    // 解析xml中设置的 android:id 属性
    mID = a.getResourceId(R.styleable.ViewStub_id, NO_ID);
    a.recycle();
    // view不可见
    setVisibility(GONE);
    // 不会调用 onDraw 方法绘制内容
    setWillNotDraw(true);
}
```

1. 获取android:inflatedId、android:layout、android:id的值
2. 调用setVisibility方法，设置View不可见
3. 调用setWillNotDraw方法，不会调用 onDraw 方法绘制内容

在上面提到了如果想要加载ViewStub所指定的layout资源，需要设置ViewStub控件设置可见，或者调用inflate()方法，来看一下ViewStub的setVisibility方法

### 1.3 ViewStub的setVisibility方法

setVisibility(int visibility)方法，参数visibility对应三个值分别是INVISIBLE、VISIBLE、GONE

* VISIBLE：视图可见
* INVISIBLE：视图不可见的，它仍然占用布局的空间
* GONE：视图不可见，它不占用布局的空间

接下里查看一下ViewStub的setVisibility方法
**frameworks/base/core/java/android/view/ViewStub.java**

```
@Override
public void setVisibility(int visibility) {
    if (mInflatedViewRef != null) {
        // mInflatedViewRef 是 WeakReference的实例，调用inflate方法时候初始化
        View view = mInflatedViewRef.get();
        if (view != null) {
            // 设置View可见
            view.setVisibility(visibility);
        } else {
            throw new IllegalStateException("setVisibility called on un-referenced view");
        }
    } else {
        super.setVisibility(visibility);
        // 当View为空且设置视图可见(VISIBLE、INVISIBLE)，调用inflate方法
        if (visibility == VISIBLE || visibility == INVISIBLE) {
            inflate();
        }
    }
}
```

1. mInflatedViewRef 是 WeakReference的实例，调用inflate方法时候初始化
2. 从 mInflatedViewRef 缓存中获取View，并且设置View可见
3. 当View为空且设置视图可见(VISIBLE、INVISIBLE)，会调用inflate方法

### 1.4 ViewStub.inflate方法

调用了ViewStub的setVisibility方法，最后都会调用ViewStub.inflate方法，来查看一下
**frameworks/base/core/java/android/view/ViewStub.java**

```
public View inflate() {
    final ViewParent viewParent = getParent();

    if (viewParent != null && viewParent instanceof ViewGroup) {
        if (mLayoutResource != 0) {
            final ViewGroup parent = (ViewGroup) viewParent;
            // 解析布局视图
            // 返回的view是android:layout指定的布局文件最顶层的view
            final View view = inflateViewNoAdd(parent);
            
            // 移除ViewStub
            // 添加view到被移除的ViewStub的位置
            replaceSelfWithView(view, parent);

            // 添加view到 mInflatedViewRef 中
            mInflatedViewRef = new WeakReference<>(view);
            if (mInflateListener != null) {
                // 加载完成之后，回调onInflate 方法
                mInflateListener.onInflate(this, view);
            }

            return view;
        } else {
            // 需要在xml中设置android:layout，不是layout，否则会抛出异常
            throw new IllegalArgumentException("ViewStub must have a valid layoutResource");
        }
    } else {
        // ViewStub不能作为根布局，它需要放在ViewGroup中, 否则会抛出异常
        throw new IllegalStateException("ViewStub must have a non-null ViewGroup viewParent");
    }
}
```

1. 调用inflateViewNoAdd方法返回android:layout指定的布局文件最顶层的view
2. 调用replaceSelfWithView方法, 移除ViewStub, 添加view到被移除的ViewStub的位置
3. 添加view到 mInflatedViewRef 中
4. 加载完成之后，回调onInflate 方法

**需要注意以下两点：**

* 使用ViewStub需要在xml中设置android:layout，不是layout，否则会抛出异常
* ViewStub不能作为根布局，它需要放在ViewGroup中, 否则会抛出异常

来查看一下inflateViewNoAdd方法和replaceSelfWithView方法

### 1.5 ViewStub.inflateViewNoAdd方法

调用inflateViewNoAdd方法返回android:layout指定的布局文件最顶层的view
**frameworks/base/core/java/android/view/ViewStub.java**

```
private View inflateViewNoAdd(ViewGroup parent) {
    final LayoutInflater factory;

    // mInflater 是View被创建的时候，如果是ViewStub, 将LayoutInflater赋值给mInflater
    if (mInflater != null) {
        factory = mInflater;
    } else {
        // 如果mInflater为空，则创建LayoutInflater
        factory = LayoutInflater.from(mContext);
    }

    // 从指定的 mLayoutResource 资源中解析布局视图
    // mLayoutResource 是在xml设置的 Android:layout 指定的布局文件
    final View view = factory.inflate(mLayoutResource, parent, false);

    // mInflatedId 是在xml设置的 inflateId
    if (mInflatedId != NO_ID) {
        // 将id复制给view
        view.setId(mInflatedId);
        //注意：如果指定了mInflatedId , 被inflate的layoutView的id就是mInflatedId
    }
    return view;
}
```

* mInflater是View被创建的时候，如果是ViewStub, 将LayoutInflater赋值给mInflater
* 如果mInflater为空则通过LayoutInflater.from(mContext)构建LayoutInflater
* 调用LayoutInflater的inflate方法解析布局视图
* 将mInflatedId设置View

### 1.6 ViewStub.replaceSelfWithView方法

调用replaceSelfWithView方法, 移除ViewStub, 添加view到被移除的ViewStub的位置
**frameworks/base/core/java/android/view/ViewStub.java**

```
private void replaceSelfWithView(View view, ViewGroup parent) {
    // 获取ViewStub在视图中的位置
    final int index = parent.indexOfChild(this);
    // 移除ViewStub
    // 注意：调用removeViewInLayout方法之后，调用findViewById()是找不到该ViewStub对象
    parent.removeViewInLayout(this);

    final ViewGroup.LayoutParams layoutParams = getLayoutParams();
    // 将xml中指定的 android:layout 布局文件中最顶层的View，添加到被移除的 ViewStub的位置
    if (layoutParams != null) {
        parent.addView(view, index, layoutParams);
    } else {
        parent.addView(view, index);
    }
}
```

1. 获取ViewStub在视图中的位置，然后移除ViewStub
3. 添加android:layout 布局文件中最顶层的View到被移除的ViewStub的位置

### 1.7 ViewStub的注意事项

* 使用ViewStub需要在xml中设置android:layout，不是layout，否则会抛出异常

```
throw new IllegalArgumentException("ViewStub must have a valid layoutResource");
```

* ViewStub不能作为根布局，它需要放在ViewGroup中, 否则会抛出异常

```
throw new IllegalStateException("ViewStub must have a non-null ViewGroup viewParent");
```

* 一旦调用setVisibility(View.VISIBLE)或者inflate()方法之后，该ViewStub将会从试图中被移除（此时调用findViewById()是找不到该ViewStub对象).

```
// 获取ViewStub在视图中的位置
final int index = parent.indexOfChild(this);
// 移除ViewStub
// 注意：调用removeViewInLayout方法之后，调用findViewById()是找不到该ViewStub对象
parent.removeViewInLayout(this);
```

* 如果指定了mInflatedId , 被inflate的layoutView的id就是mInflatedId

```
// mInflatedId 是在xml设置的 inflateId
if (mInflatedId != NO_ID) {
    // 将id复制给view
    view.setId(mInflatedId);
    //注意：如果指定了mInflatedId , 被inflate的layoutView的id就是mInflatedId
}
```

* 被inflate的layoutView的layoutParams与ViewStub的layoutParams相同.

```
final ViewGroup.LayoutParams layoutParams = getLayoutParams();
// 将xml中指定的 android:layout 布局文件中最顶层的View 也就是根view，
// 添加到被移除的 ViewStub的位置
if (layoutParams != null) {
    parent.addView(view, index, layoutParams);
} else {
    parent.addView(view, index);
}
```

到这里关于ViewStub的构建、布局的加载以及注意事项分析完了，接下来分析一下 LayoutInflater是如何被创建的

## 2 关于LayoutInflater

在[0xA02 Android 10 源码分析：Apk加载流程之资源加载](https://juejin.im/post/5e6c8c14f265da574b792a1a)文章中，介绍了Activity启动的时候通过调用LayoutInflater的inflater的方法加载layout文件，那么LayoutInflater是如何被创建的呢，先来看一段代码，相信下面的代码都不会很陌生

```
public RecyclerView.ViewHolder onCreateViewHolder(@NonNull ViewGroup viewGroup, int i) {
    return new AllVh(LayoutInflater
            .from(viewGroup.getContext())
            .inflate(R.layout.list_item, viewGroup, false));
}
```

**LayoutInflater的inflate方法的三个参数都代表什么意思？**

* resource：要解析的xml布局文件Id
* root：表示根布局
* attachToRoot：是否要添加到父布局root中

resource其实很好理解就是资源Id，而root 和 attachToRoot 分别代表什么意思：

* 当attachToRoot == true且root ！= null时，新解析出来的View会被add到root中去，然后将root作为结果返回
* 当attachToRoot == false且root ！= null时，新解析的View会直接作为结果返回，而且root会为新解析的View生成LayoutParams并设置到该View中去
* 当attachToRoot == false且root == null时，新解析的View会直接作为结果返回

### 2.1 LayoutInflater是如何被创建的

LayoutInflater是一个抽象类，通过调用了from()的静态函数，经由系统服务LAYOUT_INFLATER_SERVICE，最终创建了一个LayoutInflater的子类对象PhoneLayoutInflater，继承结构如下：

![](http://cdn.51git.cn/2020-03-28-15837476773367.jpg)

LayoutInflater.from(ctx) 就是根据传递过来的Context对象，调用getSystemService()来获取对应的系统服务, 来看一下这个方法
**frameworks/base/core/java/android/view/LayoutInflater.java**

```
public static LayoutInflater from(Context context) {
    // 获取系统服务 LAYOUT_INFLATER_SERVICE ，并赋值给 LayoutInflater
    // Context 是一个抽象类，真正的实现类是ContextImpl
    LayoutInflater LayoutInflater =
            (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
    if (LayoutInflater == null) {
        throw new AssertionError("LayoutInflater not found.");
    }
    return LayoutInflater;
}
```

而Context本身是一个抽象类，它真正的实例化对象是ContextImpl
**frameworks/base/core/java/android/app/ContextImpl.java**

```
public Object getSystemService(String name) {
    // SystemServiceRegistry 是管理系统服务的
    // 调用getSystemService方法，通过服务名字查找对应的服务
    return SystemServiceRegistry.getSystemService(this, name);
}
```

### 2.2 SystemServiceRegistry

SystemServiceRegistry管理所有的系统服务，调用getSystemService方法，通过服务名字查找对应的服务
**frameworks/base/core/java/android/app/SystemServiceRegistry.java**

```
public static Object getSystemService(ContextImpl ctx, String name) {
    // SYSTEM_SERVICE_FETCHERS 是一个map集合
    // 从 map 集合中取出系统服务
    ServiceFetcher<?> fetcher = SYSTEM_SERVICE_FETCHERS.get(name);
    return fetcher != null ? fetcher.getService(ctx) : null;
}
```

* ServiceFetcher为SystemServiceRegistry类的静态内部接口，定义了getService方法
* ServiceFetcher的实现类CachedServiceFetcher实现了getService方法
* 所有的系统服务都存储在一个map集合SYSTEM_SERVICE_FETCHERS当中，调用get方法来获取对应的服务

如果有getSystemService方法来获取服务，那么相应的也会有添加服务的方法
**frameworks/base/core/java/android/app/SystemServiceRegistry.java**

```
private static <T> void registerService(String serviceName, Class<T> serviceClass,
        ServiceFetcher<T> serviceFetcher) {
    SYSTEM_SERVICE_NAMES.put(serviceClass, serviceName);
    SYSTEM_SERVICE_FETCHERS.put(serviceName, serviceFetcher);
}
```

通过调用SYSTEM_SERVICE_NAMES的put方法，往map集合中添加数据，那么registerService是什么时候调用的，在SystemServiceRegistry类中搜索registerService方法，知道了在类加载的时候通过静态代码块中添加的，来看一下

```
static {
    // 初始化加载所有的系统服务

    ...
    // 省略了很多系统服务

    registerService(Context.LAYOUT_INFLATER_SERVICE, LayoutInflater.class,
            new CachedServiceFetcher<LayoutInflater>() {
        @Override
        public LayoutInflater createService(ContextImpl ctx) {
            return new PhoneLayoutInflater(ctx.getOuterContext());
        }});

    ...
    // 省略了很多系统服务
}
```

最终是创建了一个PhoneLayoutInflater并返回的，到这里LayoutInflater的创建流程就分析完了

## 总结

**View中的INVISIBLE、VISIBLE、GONE都有什么作用？**

如果想隐藏或者显示View，可以通过调用setVisibility(int visibility)方法来实现，参数visibility对应三个值分别是INVISIBLE、VISIBLE、GONE

* VISIBLE：视图可见
* INVISIBLE：视图不可见的，它仍然占用布局的空间
* GONE：视图不可见，它不占用布局的空间

**为什么ViewStub是大小为0的视图？**
**frameworks/base/core/java/android/view/ViewStub.java**

```
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    // 设置视图大小为0
    setMeasuredDimension(0, 0);
}
```

**ViewStub有什么作用？**

ViewStub的作用主要用来延迟布局的加载

**ViewStub是如何创建的？**

因为ViewStub是继承View, 所以ViewStub的创建和View的创建是相同的，通过调用了LayoutInflater.createView方法根据完整的类的路径名利用反射机制构建View对象
**frameworks/base/core/java/android/view/LayoutInflater.java**

```
...

try {
    // 利用构造函数，创建View
    final View view = constructor.newInstance(args);
    if (view instanceof ViewStub) {
        // 如果是ViewStub，则设置LayoutInflater
        final ViewStub viewStub = (ViewStub) view;
        viewStub.setLayoutInflater(cloneInContext((Context) args[0]));
    }
    return view;
} finally {
    mConstructorArgs[0] = lastContext;
}

...
```

**为什么ViewStub能做到延迟加载？**

因为在解析layout文件过程中遇到ViewStub，只是构建ViewStub的对象和初始化ViewStub的属性，没有真正开始解析view，所以可以做到延迟初始化

**ViewStub指定的Layout布局文件是什么时候被加载的？**

当ViewStub控件设置可见，或者调用inflate()方法，ViewStub所指定的layout资源就会被加载

**LayoutInflater是一个抽象类它如何被创建的？**

LayoutInflater是一个抽象类，通过调用了from()的静态函数，经由系统服务LAYOUT_INFLATER_SERVICE，最终创建了一个LayoutInflater的子类对象PhoneLayoutInflater，继承结构如下：

![](http://cdn.51git.cn/2020-03-28-15837476773367.jpg)

LayoutInflater.from(ctx) 就是根据传递过来的Context对象，调用getSystemService()来获取对应的系统服务

**系统服务存储在哪里？如何获取和添加系统服务？**

SystemServiceRegistry管理所有的系统服务，所有的系统服务都存储在一个map集合SYSTEM_SERVICE_FETCHERS当中，调用getSystemService方法获取系统服务，调用registerService方法添加系统服务

## 结语

致力于分享一系列 Android 系统源码、逆向分析、算法、翻译、Jetpack  源码相关的文章，如果你同我一样喜欢研究 Android 源码，可以关注我，如果你喜欢这篇文章欢迎 star，一起来学习，期待与你一起成长

**算法**

由于 LeetCode 的题库庞大，每个分类都能筛选出数百道题，由于每个人的精力有限，不可能刷完所有题目，因此我按照经典类型题目去分类、和题目的难易程度去排序

* 数据结构： 数组、栈、队列、字符串、链表、树……
* 算法： 查找算法、搜索算法、位运算、排序、数学、……

每道题目都会用 Java 和 kotlin 去实现，并且每道题目都有解题思路，如果你同我一样喜欢算法、LeetCode，可以关注我 GitHub 上的 LeetCode 题解：[Leetcode-Solutions-with-Java-And-Kotlin](https://github.com/hi-dhl/Leetcode-Solutions-with-Java-And-Kotlin)，一起来学习，期待与你一起成长








