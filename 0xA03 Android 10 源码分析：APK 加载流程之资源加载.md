# 0xA03 Android 10 源码分析：APK 加载流程之资源加载

![article-header-c8baf0e](http://cdn.51git.cn/2020-05-05-article-header-c8baf0ec.png)

## 引言

> * 这是 Android 10 源码分析系列的第 3 篇
> * 分支：android-10.0.0_r14
> * 全文阅读大概 15 分钟

**通过这篇文章你将学习到以下内容，文末会给出相应的答案**

* LayoutInflater的inflate 方法的三个参数都代表什么意思？
* 系统对 merge、include 是如何处理的
* merge 标签为什么可以起到优化布局的效果？
* XML 中的 View 是如何被实例化的？
* 为什么复杂布局会产生卡顿？在 Android 10 上做了那些优化？
* BlinkLayout 是什么？

前面两篇文章 [0xA01 Android 10 源码分析：APK 是如何生成的](https://juejin.im/post/5e4366c3f265da57397e1189) 和 [0xA02 Android 10 源码分析：APK 的安装流程](https://juejin.im/post/5e5a1e6a6fb9a07cb427d8cd) 分析了 APK 大概可以分为代码和资源两部分，那么 APK 的加载也是分为代码和资源两部分，代码的加载涉及了进程的创建、启动、调度，本文主要来分析一下资源的加载，如果没有看过 **APK 是如何生成的** 和 **APK 的安装流程** 可以点击下方连接前往：

* [0xA01 Android 10 源码分析：APK 是如何生成的](https://juejin.im/post/5e4366c3f265da57397e1189)
* [0xA02 Android 10 源码分析：APK 的安装流程](https://juejin.im/post/5e5a1e6a6fb9a07cb427d8cd)

## 1. Android 资源

Android 资源大概分为两个部分：assets 和 res

**assets 资源**

assets 资源放在 assets 目录下，它里面保存一些原始的文件，可以以任何方式来进行组织，这些文件最终会原封不动的被打包进 APK 文件中，通过AssetManager 来获取 asset 资源，代码如下

```
AssetManager assetManager = context.getAssets();
InputStream is = assetManager.open("fileName");
```

**res资源**

res 资源放在主工程的 res 目录下，这类资源一般都会在编译阶段生成一个资源 ID 供我们使用，res 目录包括 animator、anim、 color、drawable、layout、menu、raw、values、XML等，通过 getResource() 去获取 Resources 对象

```
Resources res = getContext().getResources();
```

APK 的生成过程中，会生成资源索引表 resources.arsc 文件和 R.java 文件，前者资源索引表 resources.arsc 记录了所有的应用程序资源目录的信息，包括每一个资源名称、类型、值、ID以及所配置的维度信息，后者定义了各个资源 ID 常量，运行时通过 Resources 和 AssetManger 共同完成资源的加载，如果资源是个文件，Resouces 先根据资源 ID 查找出文件名，AssetManger 再根据文件名查找出具体的资源，关于 resources.arsc，可以查看 [0xA01 ASOP应用框架：APK 是如何生成的](mweblib://15812319114373)<br/>

## 2. 资源的加载和解析到 View 的生成

下面代码一定不会很陌生，在 Activity 常见的几行代码

```
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.main_activity)
}
```

一起来分析一下调用 setContentView 方法之后做了什么事情，接下来查看一下 Activity 中的 setContentView 方法
**frameworks/base/core/java/android/app/Activity.java**

```
public void setContentView(@LayoutRes int layoutResID) {
    // 实际上调用的是PhoneWindow.setContentView方法
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
```

调用 getWindow 方法返回的是 mWindow，mWindow 是 Windowd 对象，实际上是调用它的唯一实现类 PhoneWindow.setContentView 方法

### 2.1 Activity -> PhoneWindow

PhoneWindow 是 Window 的唯一实现类，它的结构如下：

![](http://cdn.51git.cn/2020-03-11-15837483023427.jpg)

当调用 Activity.setContentView 方法实际上调用的是 PhoneWindow.setContentView 方法
**frameworks/base/core/java/com/android/internal/policy/PhoneWindow.java**

```
public void setContentView(int layoutResID) {
    // mContentParent是ID为ID_ANDROID_CONTENT的FrameLayout
    // 调用setContentView方法，就是给ID为ID_ANDROID_CONTENT的View添加子View
    if (mContentParent == null) {
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        // FEATURE_CONTENT_TRANSITIONS，则是标记当前内容加载有没有使用过度动画
        // 如果内容已经加载过，并且不需要动画，则会调用removeAllViews
        mContentParent.removeAllViews();
    }

    // 检查是否设置了FEATURE_CONTENT_TRANSITIONS
    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                getContext());
        transitionTo(newScene);
    } else {
        // 解析指定的XML资源文件
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    mContentParent.requestApplyInsets();
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        cb.onContentChanged();
    }
    mContentParentExplicitlySet = true;
}
```

* 先判断 mContentParent 是否为空，如果为空则调用 installDecor 方法，生成 mDecor，并将它赋值给 mContentParent
* 根据 FEATURE_CONTENT_TRANSITIONS 标记来判断是否加载过转场动画
* 如果设置了 FEATURE_CONTENT_TRANSITIONS 则添加 Scene 来过度启动，否则调用 mLayoutInflater.inflate(layoutResID, mContentParent)，解析资源文件，创建 View, 并添加到 mContentParent 视图中


### 2.2 PhoneWindow -> LayoutInflater

当调用 PhoneWindow.setContentView 方法，之后调用 LayoutInflater.inflate 方法，来解析 XML 资源文件
**frameworks/base/core/java/android/view/LayoutInflater.java**

```
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
    return inflate(resource, root, root != null);
}
```

inflate 它有多个重载方法，最后调用的是 inflate(resource, root, root != null) 方法
**frameworks/base/core/java/android/view/LayoutInflater.java**

```
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
    final Resources res = getContext().getResources();
    // 根据XML预编译生成compiled_view.dex, 然后通过反射来生成对应的View，从而减少XmlPullParser解析Xml的时间
    // 需要注意的是在目前的release版本中不支持使用
    View view = tryInflatePrecompiled(resource, res, root, attachToRoot);
    if (view != null) {
        return view;
    }
    // 获取资源解析器 XmlResourceParser
    XmlResourceParser parser = res.getLayout(resource);
    try {
        return inflate(parser, root, attachToRoot);
    } finally {
        parser.close();
    }
}
```

这个方法主要做了三件事：
* 根据 XML 预编译生成 compiled_view.dex, 然后通过反射来生成对应的 View
* 获取 XmlResourceParser
* 解析 View

**注意：在目前的 release 版本中不支持使用 tryInflatePrecompiled 方法源码如下：**

```
private void initPrecompiledViews() {
    // Precompiled layouts are not supported in this release.
    // enabled 是否启动预编译布局，这里始终为false
    boolean enabled = false;
    initPrecompiledViews(enabled);
}

private void initPrecompiledViews(boolean enablePrecompiledViews) {
    mUseCompiledView = enablePrecompiledViews;

    if (!mUseCompiledView) {
        mPrecompiledClassLoader = null;
        return;
    }

    ...
}

View tryInflatePrecompiled(@LayoutRes int resource, Resources res, @Nullable ViewGroup root,
    boolean attachToRoot) {
    // mUseCompiledView始终为false
    if (!mUseCompiledView) {
        return null;
    }

    // 获取需要解析的资源文件的 pkg 和 layout
    String pkg = res.getResourcePackageName(resource);
    String layout = res.getResourceEntryName(resource);

    try {
        // 根据mPrecompiledClassLoader通过反射获取预编译生成的view对象的Class类
        Class clazz = Class.forName("" + pkg + ".CompiledView", false, mPrecompiledClassLoader);
        Method inflater = clazz.getMethod(layout, Context.class, int.class);
        View view = (View) inflater.invoke(null, mContext, resource);

        if (view != null && root != null) {
            // 将生成的view 添加根布局中
            XmlResourceParser parser = res.getLayout(resource);
            try {
                AttributeSet attrs = Xml.asAttributeSet(parser);
                advanceToRootNode(parser);
                ViewGroup.LayoutParams params = root.generateLayoutParams(attrs);
                // 如果 attachToRoot=true添加到根布局中
                if (attachToRoot) {
                    root.addView(view, params);
                } else {
                    // 否者将获取到的根布局的LayoutParams，设置到生成的view中
                    view.setLayoutParams(params);
                }
            } finally {
                parser.close();
            }
        }

        return view;
    } catch (Throwable e) {
        
    } finally {
    }
    return null;
}
```

* tryInflatePrecompiled 方法是 Android 10 新增的方法，这是一个在编译器运行的一个优化，因为布局文件越复杂 XmlPullParser 解析 XML 越耗时, tryInflatePrecompiled 方法根据 XML 预编译生成compiled_view.dex, 然后通过反射来生成对应的 View，从而减少 XmlPullParser 解析 XML 的时间，然后根据 attachToRoot 参数来判断是添加到根布局中，还是设置 LayoutParams 参数返回给调用者
* 用一个全局变量 mUseCompiledView 来控制是否启用 tryInflatePrecompiled 方法，根据源码分析，mUseCompiledView 始终为 false

**了解了 tryInflatePrecompiled 方法之后，在来查看一下 inflate 方法中的三个参数都什么意思**

* resource：要解析的 XML 布局文件 ID
* root：表示根布局
* attachToRoot：是否要添加到父布局 root 中

resource 其实很好理解就是资源 ID，而 root 和 attachToRoot 分别代表什么意思：

* 当 attachToRoot == true 且 root ！= null 时，新解析出来的 View 会被 add 到 root 中去，然后将 root 作为结果返回
* 当 attachToRoot == false 且 root ！= null 时，新解析的 View 会直接作为结果返回，而且 root 会为新解析的 View 生成 LayoutParams 并设置到该 View 中去
* 当 attachToRoot == false 且 root == null 时，新解析的 View 会直接作为结果返回

根据源码知道调用 tryInflatePrecompiled 方法返回的 view 为空，继续往下执行调用 Resources 的 getLayout 方法获取资源解析器  XmlResourceParser

### 2.3 LayoutInflater -> Resources

上面说到 XmlResourceParser 是通过调用 Resources 的 getLayout 方法获取的，getLayout 方法又去调用了 Resources 的loadXmlResourceParser 方法
**frameworks/base/core/java/android/content/res/Resources.java**

```
public XmlResourceParser getLayout(@LayoutRes int id) throws NotFoundException {
    return loadXmlResourceParser(id, "layout");
}

XmlResourceParser loadXmlResourceParser(@AnyRes int id, @NonNull String type)
        throws NotFoundException {
    // TypedValue 主要用来存储资源
    final TypedValue value = obtainTempTypedValue();
    try {
        final ResourcesImpl impl = mResourcesImpl;
        // 获取XML资源，保存到 TypedValue
        impl.getValue(id, value, true);
        if (value.type == TypedValue.TYPE_STRING) {
            // 为指定的XML资源，加载解析器
            return impl.loadXmlResourceParser(value.string.toString(), id,
                    value.assetCookie, type);
        }
        throw new NotFoundException("Resource ID #0x" + Integer.toHexString(id)
                + " type #0x" + Integer.toHexString(value.type) + " is not valid");
    } finally {
        releaseTempTypedValue(value);
    }
}
```

TypedValue 是动态的数据容器，主要用来存储 Resource 的资源，获取 XML 资源保存到 TypedValue，之后调用 ResourcesImpl 的 loadXmlResourceParser 方法加载对应的解析器

### 2.4 Resources -> ResourcesImpl

ResourcesImpl 实现了 Resource 的访问，它包含了 AssetManager 和所有的缓存，通过 Resource 的 getValue 方法获取 XML 资源保存到  TypedValue，之后就会调用 ResourcesImpl 的 loadXmlResourceParser 方法对该布局资源进行解析
**frameworks/base/core/java/android/content/res/ResourcesImpl.java**

```
XmlResourceParser loadXmlResourceParser(@NonNull String file, @AnyRes int id, int assetCookie,
        @NonNull String type)
        throws NotFoundException {
    if (id != 0) {
        try {
            synchronized (mCachedXmlBlocks) {
                final int[] cachedXmlBlockCookies = mCachedXmlBlockCookies;
                final String[] cachedXmlBlockFiles = mCachedXmlBlockFiles;
                final XmlBlock[] cachedXmlBlocks = mCachedXmlBlocks;
                // 首先从缓存中查找XML资源
                final int num = cachedXmlBlockFiles.length;
                for (int i = 0; i < num; i++) {
                    if (cachedXmlBlockCookies[i] == assetCookie && cachedXmlBlockFiles[i] != null
                            && cachedXmlBlockFiles[i].equals(file)) {
                        // 调用newParser方法去构建一个XmlResourceParser对象，返回给调用者
                        return cachedXmlBlocks[i].newParser(id);
                    }
                }

                // 如果缓存中没有，则创建XmlBlock，并将它放到缓存中
                // XmlBlock是已编译的XML文件的一个包装类
                final XmlBlock block = mAssets.openXmlBlockAsset(assetCookie, file);
                if (block != null) {
                    final int pos = (mLastCachedXmlBlockIndex + 1) % num;
                    mLastCachedXmlBlockIndex = pos;
                    final XmlBlock oldBlock = cachedXmlBlocks[pos];
                    if (oldBlock != null) {
                        oldBlock.close();
                    }
                    cachedXmlBlockCookies[pos] = assetCookie;
                    cachedXmlBlockFiles[pos] = file;
                    cachedXmlBlocks[pos] = block;
                    // 调用newParser方法去构建一个XmlResourceParser对象，返回给调用者
                    return block.newParser(id);
                }
            }
        } catch (Exception e) {
            final NotFoundException rnf = new NotFoundException("File " + file
                    + " from xml type " + type + " resource ID #0x" + Integer.toHexString(id));
            rnf.initCause(e);
            throw rnf;
        }
    }

    throw new NotFoundException("File " + file + " from xml type " + type + " resource ID #0x"
            + Integer.toHexString(id));
}
```

首先从缓存中查找 XML 资源之后调用 newParser 方法，如果缓存中没有，则调用 AssetManger 的 openXmlBlockAsset 方法创建一个 XmlBlock，并将它放到缓存中，XmlBlock 是已编译的 XML 文件的一个包装类
**frameworks/base/core/java/android/content/res/AssetManager.java**

```
XmlBlock openXmlBlockAsset(int cookie, @NonNull String fileName) throws IOException {
    Preconditions.checkNotNull(fileName, "fileName");
    synchronized (this) {
        ensureOpenLocked();
        // 调用native方法nativeOpenXmlAsset, 加载指定的XML资源文件，得到ResXMLTree
        // xmlBlock是ResXMLTree对象的地址
        final long xmlBlock = nativeOpenXmlAsset(mObject, cookie, fileName);
        if (xmlBlock == 0) {
            throw new FileNotFoundException("Asset XML file: " + fileName);
        }
        // 创建XmlBlock，封装xmlBlock，返回给调用者
        final XmlBlock block = new XmlBlock(this, xmlBlock);
        incRefsLocked(block.hashCode());
        return block;
    }
}
```

最终调用 native 方法 nativeOpenXmlAsset 去打开指定的 XML 文件，加载对应的资源，来查看一下 navtive 方法 NativeOpenXmlAsset
**frameworks/base/core/jni/android_util_AssetManager.cpp**

```
// java方法对应的native方法
{"nativeOpenXmlAsset", "(JILjava/lang/String;)J", (void*)NativeOpenXmlAsset}
    
static jlong NativeOpenXmlAsset(JNIEnv* env, jobject /*clazz*/, jlong ptr, jint jcookie,
                                jstring asset_path) {
  ApkAssetsCookie cookie = JavaCookieToApkAssetsCookie(jcookie);
  ...
  
  const DynamicRefTable* dynamic_ref_table = assetmanager->GetDynamicRefTableForCookie(cookie);

  std::unique_ptr<ResXMLTree> xml_tree = util::make_unique<ResXMLTree>(dynamic_ref_table);
  status_t err = xml_tree->setTo(asset->getBuffer(true), asset->getLength(), true);
  asset.reset();
  ...
  
  return reinterpret_cast<jlong>(xml_tree.release());
}
```

* C++ 层的 NativeOpenXmlAsset 方法会创建 ResXMLTree 对象，返回的是 ResXMLTree 在 C++ 层的地址
* Java 层 nativeOpenXmlAsse t方法的返回值 xmlBlock 是 C++ 层的 ResXMLTree 对象的地址，然后将 xmlBlock 封装进 XmlBlock 中返回给调用者

当 xmlBlock 创建之后，会调用 newParser 方法，构建一个 XmlResourceParser 对象，返回给调用者

### 2.5 ResourcesImpl -> XmlBlock

XmlBlock 是已编译的 XML 文件的一个包装类，XmlResourceParser 负责对 XML 的标签进行遍历解析的，它的真正的实现是 XmlBlock 的内部类 XmlBlock.Parser，而真正完成 XML 的遍历操作的函数都是由 XmlBlock 来实现的，为了提升效率都是通过 JNI 调用 native 的函数来做的，接下来查看一下 newParser 方法
**frameworks/base/core/java/android/content/res/XmlBlock.java**

```
public XmlResourceParser newParser(@AnyRes int resId) {
    synchronized (this) {
        // mNative是C++层的ResXMLTree对象的地址
        if (mNative != 0) {
            // nativeCreateParseState方法根据 mNative 查找到ResXMLTree，
            // 在C++层构建一个ResXMLParser对象，
            // 构建Parser，封装ResXMLParser，返回给调用者
            return new Parser(nativeCreateParseState(mNative, resId), this);
        }
        return null;
    }
}
```

这个方法做两件事
* mNative 是 C++ 层的 ResXMLTree 对象的地址，调用 native 方法 nativeCreateParseState，在 C++ 层构建一个 ResXMLParser 对象，返回 ResXMLParser 对象在 C++ 层的地址
* Java 层拿到 ResXMLParser 在 C++ 层地址，构建 Parser，封装 ResXMLParser，返回给调用者

接下来查看一下 native 方法 nativeCreateParseState
**frameworks/base/core/jni/android_util_XmlBlock.cpp**

```
// java方法对应的native方法
{ "nativeCreateParseState",     "(JI)J",
            (void*) android_content_XmlBlock_nativeCreateParseState }
            
            
static jlong android_content_XmlBlock_nativeCreateParseState(JNIEnv* env, jobject clazz,
                                                          jlong token, jint res_id)
{
    ResXMLTree* osb = reinterpret_cast<ResXMLTree*>(token);
    if (osb == NULL) {
        jniThrowNullPointerException(env, NULL);
        return 0;
    }

    ResXMLParser* st = new ResXMLParser(*osb);
    if (st == NULL) {
        jniThrowException(env, "java/lang/OutOfMemoryError", NULL);
        return 0;
    }

    st->setSourceResourceId(res_id);
    st->restart();

    return reinterpret_cast<jlong>(st);
}
```

* token 对应 Java 层 mNative，是 C++ 层的 ResXMLTree 对象的地址
* 调用 C++ 层 android_content_XmlBlock_nativeCreateParseState 方法，根据 token找到 ResXMLTree 对象
* 在 C++ 层构建一个 ResXMLParser 对象，返给 Java 层对应 ResXMLParser 对象在 C++ 层的地址
* Java 层拿到 ResXMLParser 在 C++ 层地址，封装到 Parser 中

### 2.6 再次回到 LayoutInflater

经过一系列的跳转，最后调用 XmlBlock.newParser 方法获取资源解析器  XmlResourceParser，之后回到 LayoutInflater 调用处 inflate 方法，然后调用 rInflate 方法解析 View
**frameworks/base/core/java/android/view/LayoutInflater.java**

```
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    synchronized (mConstructorArgs) {
        // 获取context
        final Context inflaterContext = mContext;
        final AttributeSet attrs = Xml.asAttributeSet(parser);
        Context lastContext = (Context) mConstructorArgs[0];
        mConstructorArgs[0] = inflaterContext;
        // 存储根布局
        View result = root;

        try {
            // 处理 START_TA G和 END_TAG
            advanceToRootNode(parser);
            final String name = parser.getName();

            // 解析merge标签，rInflate方法会将merge标签下面的所有子view添加到根布局中
            // 这也是为什么merge标签可以简化布局的效果
            if (TAG_MERGE.equals(name)) {
                if (root == null || !attachToRoot) {
                    throw new InflateException("<merge /> can be used only with a valid "
                            + "ViewGroup root and attachToRoot=true");
                }
                // 解析merge标签下的所有的View，添加到根布局中
                rInflate(parser, root, inflaterContext, attrs, false);
            } else {
                // 如果不是merge标签，调用createViewFromTag方法解析布局视图，这里的temp其实是我们xml里的top view
                final View temp = createViewFromTag(root, name, inflaterContext, attrs);
                ViewGroup.LayoutParams params = null;

                // 如果根布局不为空的话，且attachToRoot为false，为View设置布局参数
                if (root != null) {
                    // 获取根布局的LayoutParams
                    params = root.generateLayoutParams(attrs);
                    // attachToRoot为false，为View设置LayoutParams
                    if (!attachToRoot) {
                        temp.setLayoutParams(params);
                    }
                }

                // 解析当前View下面的所有子View
                rInflateChildren(parser, temp, attrs, true);

                // 如果 root 不为空且 attachToRoot 为false，将解析出来的View 添加到根布局
                if (root != null && attachToRoot) {
                    root.addView(temp, params);
                }

                // 如果根布局为空 或者 attachToRoot 为false，返回当前的View
                if (root == null || !attachToRoot) {
                    result = temp;
                }
            }

        } catch (XmlPullParserException e) {
            final InflateException ie = new InflateException(e.getMessage(), e);
            ie.setStackTrace(EMPTY_STACK_TRACE);
            throw ie;
        } catch (Exception e) {
            throw ie;
        } finally {
        }
        return result;
    }
}
```

* 解析 merge 标签，使用 merge 标签必须有父布局，且依赖于父布局加载
* rInflate 方法会将 merge 标签下面的所有 View 添加到根布局中
* 如果不是 merge 标签，调用 createViewFromTag 解析布局视图，返回 temp, 这里的 temp 其实是我们 XML 里的 Top View
* 调用 rInflateChildren 方法，传递参数 temp，在 rInflateChildren方 法里内部，会调用 rInflate 方法, 解析当前 View 下面的所有子 View

**通过分析源码知道了attachToRoot 和 root的参数代表什么意思，这里总结一下：**

* 当 attachToRoot == true 且 root ！= null 时，新解析出来的 View 会被 add 到 root 中去，然后将 root 作为结果返回
* 当 attachToRoot == false 且 root ！= null 时，新解析的 View 会直接作为结果返回，而且 root 会为新解析的View生成 LayoutParams并设置到该 View 中去
* 当 attachToRoot == false 且 root == null 时，新解析的 View 会直接作为结果返回

无论是不是 merge 标签，最后都会调用 rInflate 方法进行 View 树的解析，他们的区别在于，如果是 merge 标签传递的参数 finishInflate 是 false，如果不是 merge 标签传递的参数 finishInflate 是 true
**frameworks/base/core/java/android/view/LayoutInflater.java**

```
void rInflate(XmlPullParser parser, View parent, Context context,
        AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {

    // 获取数的深度
    final int depth = parser.getDepth();
    int type;
    boolean pendingRequestFocus = false;
    // 逐个 View 解析
    while (((type = parser.next()) != XmlPullParser.END_TAG ||
            parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

        if (type != XmlPullParser.START_TAG) {
            continue;
        }

        final String name = parser.getName();

        if (TAG_REQUEST_FOCUS.equals(name)) {
            // 解析android:focusable="true", 获取View的焦点
            pendingRequestFocus = true;
            consumeChildElements(parser);
        } else if (TAG_TAG.equals(name)) {
            // 解析android:tag标签
            parseViewTag(parser, parent, attrs);
        } else if (TAG_INCLUDE.equals(name)) {
            // 解析include标签，include标签不能作为根布局
            if (parser.getDepth() == 0) {
                throw new InflateException("<include /> cannot be the root element");
            }
            parseInclude(parser, context, parent, attrs);
        } else if (TAG_MERGE.equals(name)) {
            // merge标签必须作为根布局
            throw new InflateException("<merge /> must be the root element");
        } else {
            // 根据元素名解析，生成View
            final View view = createViewFromTag(parent, name, context, attrs);
            final ViewGroup viewGroup = (ViewGroup) parent;
            final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
            // rInflateChildren方法内部调用的rInflate方法，深度优先遍历解析所有的子View
            rInflateChildren(parser, view, attrs, true);
            // 添加解析的View
            viewGroup.addView(view, params);
        }
    }

    if (pendingRequestFocus) {
        parent.restoreDefaultFocus();
    }

    // 如果finishInflate为true，则调用onFinishInflate方法
    if (finishInflate) {
        parent.onFinishInflate();
    }
}
```

整个 View 树的解析过程如下：

* 获取 View 树的深度
* 逐个 View 解析
* 解析 android:focusable="true", 获取 View 的焦点
* 解析 android:tag 标签
* 解析 include 标签，并且 include 标签不能作为根布局
* 解析 merge 标签，并且 merge 标签必须作为根布局
* 根据元素名解析，生成对应的 View
* rInflateChildren 方法内部调用的 rInflate 方法，深度优先遍历解析所有的子 View
* 添加解析的 View

**注意：通过分析源码, 以下几点需要特别注意**

* include 标签不能作为根元素，需要放在 ViewGroup中
* merge 标签必须为根元素，使用 merge 标签必须有父布局，且依赖于父布局加载
* 当 XmlResourseParser 对 XML 的遍历，随着布局越复杂，层级嵌套越多，所花费的时间也越长，所以对布局的优化，可以使用 meger 标签减少层级的嵌套

在解析过程中调用 createViewFromTag 方法，根据元素名解析，生成对应的 View，接下来查看一下 createViewFromTag 方法
**frameworks/base/core/java/android/view/LayoutInflater.java**

```
private View createViewFromTag(View parent, String name, Context context, AttributeSet attrs) {
    return createViewFromTag(parent, name, context, attrs, false);
}

View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
        boolean ignoreThemeAttr) {
    if (name.equals("view")) {
        name = attrs.getAttributeValue(null, "class");
    }

    // 如果设置了theme, 构建一个ContextThemeWrapper
    if (!ignoreThemeAttr) {
        final TypedArray ta = context.obtainStyledAttributes(attrs, ATTRS_THEME);
        final int themeResId = ta.getResourceId(0, 0);
        if (themeResId != 0) {
            context = new ContextThemeWrapper(context, themeResId);
        }
        ta.recycle();
    }

    try {
        // 如果name是blink，则创建BlinkLayout
        // 如果设置factory，根据factory进行解析, 这是系统留给我们的Hook入口
        View view = tryCreateView(parent, name, context, attrs);

        // 如果 tryCreateView方法返回的View为空，则判断是内置View还是自定义View
        // 如果是内置的View则调用onCreateView方法，如果是自定义View 则调用createView方法
        if (view == null) {
            final Object lastContext = mConstructorArgs[0];
            mConstructorArgs[0] = context;
            try {
                // 如果使用自定义View，需要在XML指定全路径的，
                // 例如：com.hi.dhl.CustomView，那么这里就有个.了
                // 可以利用这一点判定是内置的View，还是自定义View
                if (-1 == name.indexOf('.')) {
                    // 解析内置View
                    view = onCreateView(context, parent, name, attrs);
                } else {
                    // 解析自定义View
                    view = createView(context, name, null, attrs);
                }
                /**
                 * onCreateView方法与createView方法的区别
                 * onCreateView方法：会给内置的View前面加一个前缀，例如：android.widget，最终会调用createView方法
                 * createView方法: 据完整的类的路径名利用反射机制构建View对象
                 */
            } finally {
                mConstructorArgs[0] = lastContext;
            }
        }

        return view;
    } catch (InflateException e) {
        throw e;

    } catch (ClassNotFoundException e) {
        throw ie;

    } catch (Exception e) {
        throw ie;
    }
}
```

* 解析 View 标签，如果设置了 theme, 构建一个 ContextThemeWrapper
* 调用 tryCreateView 方法，如果 name 是 blink，则创建 BlinkLayout，如果设置 factory，根据 factory 进行解析，这是系统留给我们的 Hook 入口，我们可以人为的干涉系统创建 View，添加更多的功能
* 如果 tryCreateView 方法返回的 View 为空，则分别调用 onCreateView 方法和 createView 方法，onCreateView 方法解析内置 View，createView 方法解析自定义 View

在解析过程中，会先调用 tryCreateView 方法，来看一下 tryCreateView 方法内部做了什么
**frameworks/base/core/java/android/view/LayoutInflater.java**

```
public final View tryCreateView(@Nullable View parent, @NonNull String name,
    @NonNull Context context,
    @NonNull AttributeSet attrs) {
    // BlinkLayout它是FrameLayout的子类，是LayoutInflater中的一个内部类,
    // 如果当前标签为TAG_1995，则创建一个隔500毫秒闪烁一次的BlinkLayout来承载它的布局内容
    if (name.equals(TAG_1995)) {
        // Let's party like it's 1995!
        // 源码注释也很有意思，写了Let's party like it's 1995!, 据说是为了庆祝1995年的复活节
        return new BlinkLayout(context, attrs);
    }
        
    // 如果设置factory，根据factory进行解析, 这是系统留给我们的Hook入口，我们可以人为的干涉系统创建View，添加更多的功能
    if (mFactory2 != null) {
        view = mFactory2.onCreateView(parent, name, context, attrs);
    } else if (mFactory != null) {
        view = mFactory.onCreateView(name, context, attrs);
    } else {
        view = null;
    }

    if (view == null && mPrivateFactory != null) {
        view = mPrivateFactory.onCreateView(parent, name, context, attrs);
    }

    return view;
}
```

* 如果 name 是 blink，则创建 BlinkLayout，返给调用者
* 如果设置 factory，根据 factory 进行解析, 这是系统留给我们的 Hook 入口，我们可以人为的干涉系统创建 View，添加更多的功能，例如夜间模式，将 View 返给调用者

根据刚才的分析，会先调用 tryCreateView 方法，如果这个方法返回的 View 为空，然后会调用 onCreateView 方法对内置 View 进行解析，createView 方法对自定义 View 进行解析

**onCreateView 方法与 createView 方法的有什么区别**

* onCreateView 方法：会给内置的 View 前面加一个前缀，例如： android.widget，最终会调用 createView 方法
* createView 方法: 根据完整的类的路径名利用反射机制构建 View 对象

来看一下这两个方法的实现，LayoutInflater 是一个抽象类，我们实际使用的是 PhoneLayoutInflater，它的结构如下

![](http://cdn.51git.cn/2020-03-11-15837476773367.jpg)

PhoneLayoutInflater 重写了 LayoutInflater 的 onCreatView 方法，这个方法就是给内置的 View 前面加一个前缀
**frameworks/base/core/java/com/android/internal/policy/PhoneLayoutInflater.java**

```
private static final String[] sClassPrefixList = {
    "android.widget.",
    "android.webkit.",
    "android.app."
};
    
protected View onCreateView(String name, AttributeSet attrs) throws ClassNotFoundException {
    for (String prefix : sClassPrefixList) {
        try {
            View view = createView(name, prefix, attrs);
            if (view != null) {
                return view;
            }
        } catch (ClassNotFoundException e) {
           
        }
    }

    return super.onCreateView(name, attrs);
}
```
 
onCreateView 方法会给内置的 View 前面加一个前缀，之后调用 createView 方法，真正的 View 构建还是在 LayoutInflater 的 createView 方法里完成的，createView 方法根据完整的类的路径名利用反射机制构建 View 对象
**frameworks/base/core/java/android/view/LayoutInflater.java**

```
public final View createView(@NonNull Context viewContext, @NonNull String name,
        @Nullable String prefix, @Nullable AttributeSet attrs)
        throws ClassNotFoundException, InflateException {
    ...

    try {

        if (constructor == null) {
            // 如果在缓存中没有找到构造函数，则根据完整的类的路径名利用反射机制构建View对象
            clazz = Class.forName(prefix != null ? (prefix + name) : name, false,
                    mContext.getClassLoader()).asSubclass(View.class);

            if (mFilter != null && clazz != null) {
                boolean allowed = mFilter.onLoadClass(clazz);
                if (!allowed) {
                    failNotAllowed(name, prefix, viewContext, attrs);
                }
            }
            // 利用反射机制构建clazz, 将它的构造函数存入sConstructorMap中，下次可以直接从缓存中查找
            constructor = clazz.getConstructor(mConstructorSignature);
            constructor.setAccessible(true);
            sConstructorMap.put(name, constructor);
        } else {
            // 如果从缓存中找到了缓存的构造函数
            if (mFilter != null) {
                Boolean allowedState = mFilterMap.get(name);
                if (allowedState == null) {
                    // 根据完整的类的路径名利用反射机制构建View对象
                    clazz = Class.forName(prefix != null ? (prefix + name) : name, false,
                            mContext.getClassLoader()).asSubclass(View.class);

                    ...
                } else if (allowedState.equals(Boolean.FALSE)) {
                    failNotAllowed(name, prefix, viewContext, attrs);
                }
            }
        }

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
    } catch (NoSuchMethodException e) {
        throw ie;

    } catch (ClassCastException e) {
        
        throw ie;
    } catch (ClassNotFoundException e) {
        throw e;
    } catch (Exception e) {
       
        throw ie;
    } finally {
    }
}
``` 

* 先从缓存中寻找构造函数，如果存在直接使用
* 如果没有找到根据完整的类的路径名利用反射机制构建 View 对象

到了这里关于 APK 的布局 XML 资源文件的查找和解析 -> View 的生成流程到这里就结束了

## 总结

那我们就来依次来回答上面提出的几个问题

**LayoutInflater 的 inflate 的三个参数都代表什么意思？**

* resource：要解析的 XML 布局文件 ID
* root：表示根布局
* attachToRoot：是否要添加到父布局 root 中

resource 其实很好理解就是资源 ID，而 root 和 attachToRoot 分别代表什么意思：

* 当 attachToRoot == true 且 root ！= null 时，新解析出来的 View 会被 add 到 root 中去，然后将 root 作为结果返回
* 当 attachToRoot == false 且 root ！= null 时，新解析的 View 会直接作为结果返回，而且 root 会为新解析的 View 生成 LayoutParams 并设置到该 View 中去
* 当 attachToRoot == false 且 root == null 时，新解析的 View 会直接作为结果返回

**系统对 merge、include 是如何处理的**

* 使用 merge 标签必须有父布局，且依赖于父布局加载
* merge 并不是一个 ViewGroup，也不是一个 View，它相当于声明了一些视图，等待被添加，解析过程中遇到 merge 标签会将 merge 标签下面的所有子 view 添加到根布局中
* merge 标签在 XML 中必须是根元素
* 相反的 include 不能作为根元素，需要放在一个 ViewGroup 中
* 使用 include 标签必须指定有效的 layout 属性
* 使用 include 标签不写宽高是没有关系的，会去解析被 include 的  layout

**merge 标签为什么可以起到优化布局的效果？**

解析过程中遇到 merge 标签，会调用 rInflate 方法，部分代码如下

```
// 根据元素名解析，生成对应的View
final View view = createViewFromTag(parent, name, context, attrs);
final ViewGroup viewGroup = (ViewGroup) parent;
final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
// rInflateChildren方法内部调用的rInflate方法，深度优先遍历解析所有的子View
rInflateChildren(parser, view, attrs, true);
// 添加解析的View
viewGroup.addView(view, params);
```

解析 merge 标签下面的所有子 View，然后添加到根布局中

**View 是如何被实例化的？**

View 分为系统 View 和自定义 View, 通过调用 onCreateView 与createView 方法进行不同的处理
* onCreateView 方法：会给内置的 View 前面加一个前缀，例如：android.widget，最终会调用 createView 方法
* createView 方法：根据完整的类的路径名利用反射机制构建 View 对象

**为什么复杂布局会产生卡顿？在 Android 10 上做了那些优化？**

* XmlResourseParser 对 XML 的遍历，随着布局越复杂，层级嵌套越多，所花费的时间也越长
* 调用 onCreateView 与 createView 方法是通过反射创建 View 对象导致的耗时
* 在 Android 10上，新增 tryInflatePrecompiled 方法是为了减少 XmlPullParser 解析 XML 的时间，但是用一个全局变量 mUseCompiledView 来控制是否启用 tryInflatePrecompiled 方法，根据源码分析，mUseCompiledView 始终为 false，所以 tryInflatePrecompiled 方法目前在 release 版本中不可使用

**BlinkLayout 是什么？**

BlinkLayout 继承 FrameLayout，是一种会闪烁的布局，被包裹的内容会一直闪烁，根据源码注释 Let's party like it's 1995!，BlinkLayout 是为了庆祝 1995 年的复活节, 有兴趣可以看看 [reddit](https://www.reddit.com/r/androiddev/comments/3sekn8/lets_party_like_its_1995_from_the_layoutinflater/) 上的讨论，来查看一下它的源码是如何实现的<br/>
 	 	   
```
private static class BlinkLayout extends FrameLayout {
    private static final int MESSAGE_BLINK = 0x42;
    private static final int BLINK_DELAY = 500;

    private boolean mBlink;
    private boolean mBlinkState;
    private final Handler mHandler;

    public BlinkLayout(Context context, AttributeSet attrs) {
        super(context, attrs);
        mHandler = new Handler(new Handler.Callback() {
            @Override
            public boolean handleMessage(Message msg) {
                if (msg.what == MESSAGE_BLINK) {
                    if (mBlink) {
                        mBlinkState = !mBlinkState;
                        // 每隔500ms循环调用
                        makeBlink();
                    }
                    // 触发dispatchDraw
                    invalidate();
                    return true;
                }
                return false;
            }
        });
    }

    private void makeBlink() {
        // 发送延迟消息
        Message message = mHandler.obtainMessage(MESSAGE_BLINK);
        mHandler.sendMessageDelayed(message, BLINK_DELAY);
    }

    @Override
    protected void onAttachedToWindow() {
        super.onAttachedToWindow();
        mBlink = true;
        mBlinkState = true;
        makeBlink();
    }

    @Override
    protected void onDetachedFromWindow() {
        super.onDetachedFromWindow();
        mBlink = false;
        mBlinkState = true;
        // 移除消息，避免内存泄露
        mHandler.removeMessages(MESSAGE_BLINK);
    }

    @Override
    protected void dispatchDraw(Canvas canvas) {
        if (mBlinkState) {
            super.dispatchDraw(canvas);
        }
    }
}
```

通过源码分析可以看出，BlinkLayout 通过 Handler 每隔 500ms 发送消息，在 handleMessage 中循环调用 invalidate 方法，通过调用 invalidate 方法，来触发 dispatchDraw 方法，做到一闪一闪的效果

## 参考

* [https://www.reddit.com/r/androiddev/comments/3sekn8/lets_party_like_its_1995_from_the_layoutinflater/](https://www.reddit.com/r/androiddev/comments/3sekn8/lets_party_like_its_1995_from_the_layoutinflater/)
* [https://github.com/RTFSC-Android/RTFSC/blob/master/LayoutInflater.md](https://github.com/RTFSC-Android/RTFSC/blob/master/LayoutInflater.md)
* [https://www.yuque.com/beesx/beesandroid/gd7w9o](https://www.yuque.com/beesx/beesandroid/gd7w9o)

## 结语

致力于分享一系列 Android 系统源码、逆向分析、算法、翻译、Jetpack  源码相关的文章，如果你同我一样喜欢研究 Android 源码，可以关注我，如果你喜欢这篇文章欢迎 star，一起来学习，期待与你一起成长

**算法**

由于 LeetCode 的题库庞大，每个分类都能筛选出数百道题，由于每个人的精力有限，不可能刷完所有题目，因此我按照经典类型题目去分类、和题目的难易程度去排序

* 数据结构： 数组、栈、队列、字符串、链表、树……
* 算法： 查找算法、搜索算法、位运算、排序、数学、……

每道题目都会用 Java 和 kotlin 去实现，并且每道题目都有解题思路，如果你同我一样喜欢算法、LeetCode，可以关注我 GitHub 上的 LeetCode 题解：[Leetcode-Solutions-with-Java-And-Kotlin](https://github.com/hi-dhl/Leetcode-Solutions-with-Java-And-Kotlin)，一起来学习，期待与你一起成长

