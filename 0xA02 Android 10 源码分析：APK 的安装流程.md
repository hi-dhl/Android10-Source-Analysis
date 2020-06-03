# 0xA02 Android 10 源码分析：APK 的安装流程

![android-APK-banner-670x335-1](http://cdn.51git.cn/2020-05-04-android-APK-banner-670x335-1.jpg)

## 前言

> * 这是 Android 10 源码分析系列的第 2 篇
> * 分支：android-10.0.0_r14
> * 全文阅读大概 10 分钟

上一篇文章介绍了 [0xA01 Android 10 源码分析：APK 是如何生成的](https://juejin.im/post/5e4366c3f265da57397e1189)，这篇文章接着介绍如何安装 APK，需要说一下 Android 10 及更高版本中, 安装器  PackageInstaller 源码位置有所变动<br/>

### PackageInstaller 源码所在位置

PackageInstaller 是系统内置的应用程序，用于安装和卸载应用<br/>

在 Android 9 及更低版本中，软件包安装和权限控制功能包含在  PackageInstaller 软件包 (//packages/apps/PackageInstaller) 中。在 Android 10 及更高版本中，权限控制功能位于单独的软件包 PermissionController (//packages/apps/PermissionController)，这两个软件包在 Android 10 中的位置如下图所示，更多信息点击这里前往 [Android 权限](https://source.android.google.cn/devices/tech/config?hl=zh-cn)

![package-instal](http://cdn.51git.cn/2020-02-29-package-install.png)

**Android 9 及更低版本中 ：**

软件包安装和权限控制功能源码路径：packages/apps/PackageInstaller

**Android 10 及更高版本：**

* 权限控制功能 PermissionController 源码路径：packages/apps/PermissionController/
* 安装器 PackageInstaller 源码路径：frameworks/base/packages/PackageInstaller/

### 在 Android 系统不同的目录存放不同类型的应用

* /system/framwork：保存的是资源型的应用程序，它们用来打包资源文件
* /system/app：保存系统自带的应用程序
* /data/app：保存用户安装的应用程序
* /data/data：应用数据目录
* /data/app-private：保存受DRM保护的私有应用程序
* /vendor/app：保存设备厂商提供的应用程序

### 查看 PackageInstaller 源码方式

* [AOSP-PackageInstaller](https://github.com/hi-dhl/AOSP-PackageInstaller/tree/android-10.0.0_r14): 包含了安装器 PackageInstaller(7.1.2、8.1.0、9.0.0、10.0.0) 的源码，可以切换分之查看，跟随 Android 版本更新，你永远可以看到最新的源代码

![source](http://cdn.51git.cn/2020-02-29-source.png)

* [aospxref](http://aospxref.com/)：这是一个在线查看 Android 源码网站，服务器在阿里云访问速度很快，文末有关这个网站的介绍
* [googlesource-PackageInstaller](https://android.googlesource.com/platform/frameworks/base/+/refs/tags/android-10.0.0_r14/packages/PackageInstaller/)：这是安装器 PackageInstaller 在 googlesource 上的地址

## 1. APK 的安装方式

安装 APK 主要分为以下三种场景

* 安装系统应用：系统启动后调用 PackageManagerService.main() 初始化注册解析安装工作

```
public static PackageManagerService main(Context context, Installer installer,
        boolean factoryTest, boolean onlyCore) {
    // Self-check for initial settings.
    PackageManagerServiceCompilerMapping.checkProperties();

    PackageManagerService m = new PackageManagerService(context, installer,
            factoryTest, onlyCore);
    m.enableSystemUserPackages();
    ServiceManager.addService("package", m);
    final PackageManagerNative pmn = m.new PackageManagerNative();
    ServiceManager.addService("package_native", pmn);
    return m;
}
```

* 通过 adb 安装：通过 pm 参数，调用 PM 的 runInstall 方法，进入 PackageManagerService 安装安装工作
* 通过系统安装器 PackageInstaller 进行安装：先调用 InstallStart 进行权限检查之后启动 PackageInstallActivity，调用 PackageInstallActivity 的 startInstall 方法，点击 OK 按钮后进入 PackageManagerService 完成拷贝解析安装工作

所有安装方式大致相同，最终就是回到 PackageManagerService 中，安装一个 APK 的大致流程如下：

![](http://cdn.51git.cn/2020-02-29-15829642823114.jpg)

 * 拷贝到 APK 文件到指定目录
 * 解压缩 APK，拷贝文件，创建应用的数据目录
 * 解析 APK 的 AndroidManifest.xml 文件
 * 向 Launcher 应用申请添加创建快捷方式

本文主要来分析通过安装器 PackageInstaller 安装 APK，这是用户最常用的一种方式

## 2. PackageInstaller 的入口

下面代码一定不会很陌生，这就是我们常用的安装 APK 的代码（PS: 关于静默安装我会后续分享在逆向开发相关的文章）

```
Intent intent = new Intent(Intent.ACTION_VIEW);
intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);

/*
* 自Android N开始，是通过FileProvider共享相关文件，但是Android Q对公
* 有目录 File API进行了限制，只能通过Uri来操作
*/
if(Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q){
    // filePath是通过ContentResolver得到的
    intent.setDataAndType(Uri.parse(filePath) ,"application/vnd.android.package-archive");
    intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
}else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
    intent.setFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
    Uri contentUri = FileProvider.getUriForFile(mContext, "com.dhl.file.fileProvider", file);
    intent.setDataAndType(contentUri, "application/vnd.android.package-archive");
} else {
    intent.setDataAndType(Uri.fromFile(file), "application/vnd.android.package-archive");
}
startActivity(intent);

// 需要在AndroidManifest添加权限
<uses-permission android:name="android.permission.REQUEST_INSTALL_PACKAGES" /> 
```

通过 intent.setDataAndType 方法指定 Intent 的数据类型为 application/vnd.android.package-archive，隐式匹配的 Activity 为 InstallStart：
**frameworks/base/packages/PackageInstaller/AndroidManifest.xml**

```
<activity android:name=".InstallStart"
        android:theme="@android:style/Theme.Translucent.NoTitleBar"
        android:exported="true"
        android:excludeFromRecents="true">
    <intent-filter android:priority="1">
        <action android:name="android.intent.action.VIEW" />
        <action android:name="android.intent.action.INSTALL_PACKAGE" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:scheme="content" />
        <data android:mimeType="application/vnd.android.package-archive" />
    </intent-filter>
    <intent-filter android:priority="1">
        <action android:name="android.intent.action.INSTALL_PACKAGE" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:scheme="package" />
        <data android:scheme="content" />
    </intent-filter>
    <intent-filter android:priority="1">
        <action android:name="android.content.pm.action.CONFIRM_INSTALL" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</activity>
```

* 本文分析的是 10.0 的源码，在 8.0、9.0、10.0 等等版本中隐式匹配的 Activity 是 InstallStart，7.0 隐式匹配的 Activity 是 PackageInstallerActivity
* 安装器 PackageInstaller 的入口 Activity 是 InstallStart，定义了两个 scheme：content 和 package

## 3. APK 的安装流程

通过上面方式找到了入口 Activity，下面我们来查看一下 APK 是如何安装的

### 3.1 InstallStart

主要工作：

1. 判断是否勾选“未知来源”选项，若未勾选跳转到设置安装未知来源界面
2. 对于大于等于 Android 8.0 版本，会先检查是否申请安装权限，若没有则中断安装
3. 判断 Uri 的 Scheme 协议，若是 content 则调用 InstallStaging, 若是 package 则调用 PackageInstallerActivity

当我们调用上面安装代码来安装 APK 时。会跳转到 InstallStart, 并调用它的 onCreate 方法：
**frameworks/base/packages/PackageInstaller/src/com/android/packageinstaller/InstallStart.java**


```
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    ...

    final boolean isSessionInstall =
            PackageInstaller.ACTION_CONFIRM_INSTALL.equals(intent.getAction());
    ...

    final ApplicationInfo sourceInfo = getSourceInfo(callingPackage);
    final int originatingUid = getOriginatingUid(sourceInfo);
    boolean isTrustedSource = false;
    // 判断是否勾选“未知来源”选项
    if (sourceInfo != null
            && (sourceInfo.privateFlags & ApplicationInfo.PRIVATE_FLAG_PRIVILEGED) != 0) {
        isTrustedSource = intent.getBooleanExtra(Intent.EXTRA_NOT_UNKNOWN_SOURCE, false);
    }
    if (!isTrustedSource && originatingUid != PackageInstaller.SessionParams.UID_UNKNOWN) {
        final int targetSdkVersion = getMaxTargetSdkVersionForUid(this, originatingUid);
        // 如果targetSdkVerison小于0中止安装
        if (targetSdkVersion < 0) {
            Log.w(LOG_TAG, "Cannot get target sdk version for uid " + originatingUid);
            mAbortInstall = true;

            // 如果targetSdkVersion大于等于26（8.0）, 且获取不到REQUEST_INSTALL_PACKAGES权限中止安装
        } else if (targetSdkVersion >= Build.VERSION_CODES.O && !declaresAppOpPermission(
                originatingUid, Manifest.permission.REQUEST_INSTALL_PACKAGES)) {
            Log.e(LOG_TAG, "Requesting uid " + originatingUid + " needs to declare permission "
                    + Manifest.permission.REQUEST_INSTALL_PACKAGES);
            mAbortInstall = true;
        }
    }

    ...

    // 如果设置了ACTION_CONFIRM_PERMISSIONS，则调用PackageInstallerActivity。
    if (isSessionInstall) {
        nextActivity.setClass(this, PackageInstallerActivity.class);
    } else {
        Uri packageUri = intent.getData();
        // 判断Uri的Scheme协议是否是content
        if (packageUri != null && packageUri.getScheme().equals(
                ContentResolver.SCHEME_CONTENT)) {
            // [IMPORTANT] This path is deprecated, but should still work.
            // 这个路径已经被起用了，但是仍然可以工作

            // 调用InstallStaging来拷贝file/content，防止被修改
            nextActivity.setClass(this, InstallStaging.class);
        } else if (packageUri != null && packageUri.getScheme().equals(
                PackageInstallerActivity.SCHEME_PACKAGE)) {
            // 如果Uri中包含package，则调用PackageInstallerActivity
            nextActivity.setClass(this, PackageInstallerActivity.class);
        } else {
            // Uri不合法
            Intent result = new Intent();
            result.putExtra(Intent.EXTRA_INSTALL_RESULT,
                    PackageManager.INSTALL_FAILED_INVALID_URI);
            setResult(RESULT_FIRST_USER, result);
            nextActivity = null;
        }
    }
    if (nextActivity != null) {
        startActivity(nextActivity);
    }
    finish();
}
```

根据 Uri 的 Scheme 协议，若是 content 则调用 InstallStaging，查看 InstallStaging 的 onResume方法：
**frameworks/base/packages/PackageInstaller/src/com/android/packageinstaller/InstallStaging.java**

```
@Override
protected void onResume() {
    super.onResume();
    if (mStagingTask == null) {
        if (mStagedFile == null) {
            // 创建临时文件 mStagedFile 用来存储数据
            try {
                mStagedFile = TemporaryFileManager.getStagedFile(this);
            } catch (IOException e) {
                showError();
                return;
            }
        }
        // 启动 StagingAsyncTask，并传入了content协议的Uri
        mStagingTask = new StagingAsyncTask();
        mStagingTask.execute(getIntent().getData());
    }
}
```

1. 创建临时文件 mStagedFile 用来存储数据
2. 启动 StagingAsyncTask，并传入了 content 协议的 Uri

```
private final class StagingAsyncTask extends AsyncTask<Uri, Void, Boolean> {
    @Override
    protected Boolean doInBackground(Uri... params) {
        ...
        Uri packageUri = params[0];
        try (InputStream in = getContentResolver().openInputStream(packageUri)) {
            ...
            // 将packageUri（content协议的Uri）的内容写入到mStagedFile中
            try (OutputStream out = new FileOutputStream(mStagedFile)) {
                byte[] buffer = new byte[1024 * 1024];
                int bytesRead;
                while ((bytesRead = in.read(buffer)) >= 0) {
                    // Be nice and respond to a cancellation
                    if (isCancelled()) {
                        return false;
                    }
                    out.write(buffer, 0, bytesRead);
                }
            }
        } catch (IOException | SecurityException | IllegalStateException e) {
            Log.w(LOG_TAG, "Error staging apk from content URI", e);
            return false;
        }
        return true;
    }

    @Override
    protected void onPostExecute(Boolean success) {
        if (success) {
            // 如果写入成功，调用DeleteStagedFileOnResult
            Intent installIntent = new Intent(getIntent());
            installIntent.setClass(InstallStaging.this, DeleteStagedFileOnResult.class);
            installIntent.setData(Uri.fromFile(mStagedFile));
            ...
            startActivity(installIntent);
            InstallStaging.this.finish();
        } else {
            showError();
        }
    }
}
```

1. doInBackground 方法中将 packageUri（content 协议的 Uri）的内容写入到 mStagedFile 中
2. 如果写入成功，调用 DeleteStagedFileOnResult 的 OnCreate 方法：
**frameworks/base/packages/PackageInstaller/src/com/android/packageinstaller/DeleteStagedFileOnResult.java**

```
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    if (savedInstanceState == null) {
        // 启动PackageInstallerActivity
        Intent installIntent = new Intent(getIntent());
        installIntent.setClass(this, PackageInstallerActivity.class);
        installIntent.setFlags(Intent.FLAG_ACTIVITY_NO_ANIMATION);
        startActivityForResult(installIntent, 0);
    }
}
```

经过分析 InstallStaging 主要起了中转作用，将 content 协议的 Uri 转换为 File 协议，最后跳转到 PackageInstallerActivity

### 3.2 PackageInstallerActivity

主要工作：

1. 显示安装界面
2. 初始化安装需要用的各种对象，比如 PackageManager、IPackageManager、AppOpsManager、UserManager、PackageInstaller 等等
3. 根据传递过来的 Scheme 协议做不同的处理
2. 检查是否允许、初始化安装
3. 在准备安装的之前，检查应用列表判断该应用是否已安装，若已安装则提示该应用已安装，由用户决定是否替换
4. 在安装界面，提取出 APK 中权限信息并展示出来
5. 点击 OK 按钮确认安装后，会调用 startInstall 开始安装工作

PackageInstallerActivity 才是应用安装器 PackageInstaller 真正的入口 Activity，查看它的 onCreate 方法：
**frameworks/base/packages/PackageInstaller/src/com/android/packageinstaller/PackageInstallerActivity.java**

```
protected void onCreate(Bundle icicle) {
    getWindow().addSystemFlags(SYSTEM_FLAG_HIDE_NON_SYSTEM_OVERLAY_WINDOWS);
    super.onCreate(null);
    // 初始化安装需要用到的对象
    mPm = getPackageManager();
    mIpm = AppGlobals.getPackageManager();
    mAppOpsManager = (AppOpsManager) getSystemService(Context.APP_OPS_SERVICE);
    mInstaller = mPm.getPackageInstaller();
    mUserManager = (UserManager) getSystemService(Context.USER_SERVICE);

    // 根据Uri的Scheme做不同的处理
    boolean wasSetUp = processPackageUri(packageUri);
    if (!wasSetUp) {
        return;
    }
    // 显示安装界面
    bindUi();
    // 检查是否允许安装包，如果允许则启动安装。如果不允许显示适当的对话框
    checkIfAllowedAndInitiateInstall();
}
```

主要做了对象的初始化，解析 Uri 的 Scheme，初始化界面，安装包检查等等工作，接着查看一下 processPackageUri 方法

```
private boolean processPackageUri(final Uri packageUri) {
    mPackageURI = packageUri;
    final String scheme = packageUri.getScheme();
    // 根据这个Scheme协议分别对package协议和file协议进行处理
    switch (scheme) {
        case SCHEME_PACKAGE: {
            try {
                // 通过PackageManager对象获取指定包名的包信息
                mPkgInfo = mPm.getPackageInfo(packageUri.getSchemeSpecificPart(),
                        PackageManager.GET_PERMISSIONS
                                | PackageManager.MATCH_UNINSTALLED_PACKAGES);
            } catch (NameNotFoundException e) {
            }
            if (mPkgInfo == null) {
                Log.w(TAG, "Requested package " + packageUri.getScheme()
                        + " not available. Discontinuing installation");
                showDialogInner(DLG_PACKAGE_ERROR);
                setPmResult(PackageManager.INSTALL_FAILED_INVALID_APK);
                return false;
            }
            mAppSnippet = new PackageUtil.AppSnippet(mPm.getApplicationLabel(mPkgInfo.applicationInfo),
                    mPm.getApplicationIcon(mPkgInfo.applicationInfo));
        } break;

        case ContentResolver.SCHEME_FILE: {
            // 根据packageUri创建一个新的File
            File sourceFile = new File(packageUri.getPath());
            // 解析APK得到APK的信息，PackageParser.Package存储了APK的所有信息
            PackageParser.Package parsed = PackageUtil.getPackageInfo(this, sourceFile);

            if (parsed == null) {
                Log.w(TAG, "Parse error when parsing manifest. Discontinuing installation");
                showDialogInner(DLG_PACKAGE_ERROR);
                setPmResult(PackageManager.INSTALL_FAILED_INVALID_APK);
                return false;
            }
            // 根据PackageParser.Package得到的APK信息，生成PackageInfo
            mPkgInfo = PackageParser.generatePackageInfo(parsed, null,
                    PackageManager.GET_PERMISSIONS, 0, 0, null,
                    new PackageUserState());
            mAppSnippet = PackageUtil.getAppSnippet(this, mPkgInfo.applicationInfo, sourceFile);
        } break;

        default: {
            throw new IllegalArgumentException("Unexpected URI scheme " + packageUri);
        }
    }

    return true;
}
```

主要对 Scheme 协议分别对 package 协议和 file 协议进行处理

**SCHEME_PACKAGE：**

* 在 package 协议中调用了 PackageManager.getPackageInfo 方法生成 PackageInfo，PackageInfo 是跨进程传递的包数据（activities、receivers、services、providers、permissions等等）包含 APK 的所有信息

**SCHEME_FILE：**

* 在 file 协议的处理中调用了 PackageUtil.getPackageInfo 方法，方法内部调用了 PackageParser.parsePackage() 把 APK 文件的 manifest 和签名信息都解析完成并保存在了 Package，Package 包含了该 APK 的所有信息
* 调用 PackageParser.generatePackageInfo 生成 PackageInfo


接着往下走，都解析完成之后，回到 onCreate 方法，继续调用 checkIfAllowedAndInitiateInstall 方法

```
private void checkIfAllowedAndInitiateInstall() {
    // 首先检查安装应用程序的用户限制，如果有限制并弹出弹出提示Dialog或者跳转到设置界面
    final int installAppsRestrictionSource = mUserManager.getUserRestrictionSource(
            UserManager.DISALLOW_INSTALL_APPS, Process.myUserHandle());
    if ((installAppsRestrictionSource & UserManager.RESTRICTION_SOURCE_SYSTEM) != 0) {
        showDialogInner(DLG_INSTALL_APPS_RESTRICTED_FOR_USER);
        return;
    } else if (installAppsRestrictionSource != UserManager.RESTRICTION_NOT_SET) {
        startActivity(new Intent(Settings.ACTION_SHOW_ADMIN_SUPPORT_DETAILS));
        finish();
        return;
    }

    // 判断如果允许安装未知来源或者根据Intent判断得出该APK不是未知来源
    if (mAllowUnknownSources || !isInstallRequestFromUnknownSource(getIntent())) {
        initiateInstall();
    } else {
        // 检查未知安装源限制,如果有限制弹出Dialog,显示相应的信息
        final int unknownSourcesRestrictionSource = mUserManager.getUserRestrictionSource(
                UserManager.DISALLOW_INSTALL_UNKNOWN_SOURCES, Process.myUserHandle());
        final int unknownSourcesGlobalRestrictionSource = mUserManager.getUserRestrictionSource(
                UserManager.DISALLOW_INSTALL_UNKNOWN_SOURCES_GLOBALLY, Process.myUserHandle());
        final int systemRestriction = UserManager.RESTRICTION_SOURCE_SYSTEM
                & (unknownSourcesRestrictionSource | unknownSourcesGlobalRestrictionSource);
        if (systemRestriction != 0) {
            showDialogInner(DLG_UNKNOWN_SOURCES_RESTRICTED_FOR_USER);
        } else if (unknownSourcesRestrictionSource != UserManager.RESTRICTION_NOT_SET) {
            startAdminSupportDetailsActivity(UserManager.DISALLOW_INSTALL_UNKNOWN_SOURCES);
        } else if (unknownSourcesGlobalRestrictionSource != UserManager.RESTRICTION_NOT_SET) {
            startAdminSupportDetailsActivity(
                    UserManager.DISALLOW_INSTALL_UNKNOWN_SOURCES_GLOBALLY);
        } else {
            // 处理未知来源的APK
            handleUnknownSources();
        }
    }
}
```

主要检查安装应用程序的用户限制，当 APK 文件不对或者安装有限制则调用 showDialogInner 方法，弹出 dialog 提示用户，显示相应的错误信息，来看一下都有那些错误信息

```
// Dialog identifiers used in showDialog
private static final int DLG_BASE = 0;
// package信息错误
private static final int DLG_PACKAGE_ERROR = DLG_BASE + 2;
// 存储空间不够
private static final int DLG_OUT_OF_SPACE = DLG_BASE + 3;
// 安装错误
private static final int DLG_INSTALL_ERROR = DLG_BASE + 4;
// 用户限制的未知来源
private static final int DLG_UNKNOWN_SOURCES_RESTRICTED_FOR_USER = DLG_BASE + 5;
private static final int DLG_ANONYMOUS_SOURCE = DLG_BASE + 6;
// 在wear上不支持
private static final int DLG_NOT_SUPPORTED_ON_WEAR = DLG_BASE + 7;
private static final int DLG_EXTERNAL_SOURCE_BLOCKED = DLG_BASE + 8;
// 安装限制用户使用的应用程序
private static final int DLG_INSTALL_APPS_RESTRICTED_FOR_USER = DLG_BASE + 9;
```

如果用户允许安装未知来源，会调用 initiateInstall 方法

```
private void initiateInstall() {
    String pkgName = mPkgInfo.packageName;
    // 检查设备上是否存在相同包名的APK
    String[] oldName = mPm.canonicalToCurrentPackageNames(new String[] { pkgName });
    if (oldName != null && oldName.length > 0 && oldName[0] != null) {
        pkgName = oldName[0];
        mPkgInfo.packageName = pkgName;
        mPkgInfo.applicationInfo.packageName = pkgName;
    }
    // 检查package是否已安装, 如果已经安装则显示对话框提示用户是否替换。
    try {
        mAppInfo = mPm.getApplicationInfo(pkgName,
                PackageManager.MATCH_UNINSTALLED_PACKAGES);
        if ((mAppInfo.flags&ApplicationInfo.FLAG_INSTALLED) == 0) {
            mAppInfo = null;
        }
    } catch (NameNotFoundException e) {
        mAppInfo = null;
    }
    // 初始化确认安装界面
    startInstallConfirm();
}
```

根据包名获取应用程序的信息，调用 startInstallConfirm 方法初始化安装确认界面后，当用户点击确认按钮之后发生了什么，接着查看确认按钮点击事件

```
private void bindUi() {
   ...
    // 点击确认按钮，安装APK
    mAlert.setButton(DialogInterface.BUTTON_POSITIVE, getString(R.string.install),
            (ignored, ignored2) -> {
                if (mOk.isEnabled()) {
                    if (mSessionId != -1) {
                        mInstaller.setPermissionsResult(mSessionId, true);
                        finish();
                    } else {
                        // 启动Activity来完成应用的安装
                        startInstall();
                    }
                }
            }, null);
   // 点击取消按钮，取消此次安装
    mAlert.setButton(DialogInterface.BUTTON_NEGATIVE, getString(R.string.cancel),
            (ignored, ignored2) -> {
                // Cancel and finish
                setResult(RESULT_CANCELED);
                if (mSessionId != -1) {
                    mInstaller.setPermissionsResult(mSessionId, false);
                }
                finish();
            }, null);
    setupAlert();
    mOk = mAlert.getButton(DialogInterface.BUTTON_POSITIVE);
    mOk.setEnabled(false);
}
```

当用户点击确认按钮调用了 startInstall 方法，启动子 Activity 完成 APK 的安装

```
private void startInstall() {
    // 启动子Activity来完成应用的安
    Intent newIntent = new Intent();
    newIntent.putExtra(PackageUtil.INTENT_ATTR_APPLICATION_INFO,
            mPkgInfo.applicationInfo);
    newIntent.setData(mPackageURI);
    newIntent.setClass(this, InstallInstalling.class);
    ...
    if(localLOGV) Log.i(TAG, "downloaded app uri="+mPackageURI);
    startActivity(newIntent);
    finish();
}
```

startInstall 方法用来跳转到 InstallInstalling，并关闭掉当前的 PackageInstallerActivity

### 3.3 InstallInstalling

主要工作：

1. 向包管理器发送包的信息，然后等待包管理器处理结果
2. 注册一个观察者 InstallEventReceiver，并接受安装成功和失败的回调
4. 在方法 onResume 中创建同步栈，打开安装 session，设置安装进度条

InstallInstalling 首先向包管理器发送包的信息，然后等待包管理器处理结果，并在方法 InstallSuccess 和方法 InstallFailed 进行成功和失败的处理，查看 InstallInstalling 的 onCreate 方法：
**frameworks/base/packages/PackageInstaller/src/com/android/packageinstaller/InstallInstalling.java**

```
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    ...

    // 判断安装的应用是否已经存在
    if ("package".equals(mPackageURI.getScheme())) {
        try {
            getPackageManager().installExistingPackage(appInfo.packageName);
            launchSuccess();
        } catch (PackageManager.NameNotFoundException e) {
            launchFailure(PackageManager.INSTALL_FAILED_INTERNAL_ERROR, null);
        }
    } else {
        final File sourceFile = new File(mPackageURI.getPath());
        PackageUtil.AppSnippet as = PackageUtil.getAppSnippet(this, appInfo, sourceFile);
        ...

        if (savedInstanceState != null) {
            // 如果savedInstanceState 不为空，获取已经存在mSessionId 和mInstallId 重新注册
            mSessionId = savedInstanceState.getInt(SESSION_ID);
            mInstallId = savedInstanceState.getInt(INSTALL_ID);
            try {
                // 根据mInstallId向InstallEventReceiver注册一个观察者，launchFinishBasedOnResult会接收到安装事件的回调
                InstallEventReceiver.addObserver(this, mInstallId,
                        this::launchFinishBasedOnResult);
            } catch (EventResultPersister.OutOfIdsException e) {
            }
        } else {
            // 如果为空创建SessionParams，代表安装会话的参数
            // 解析APK, 并将解析的参数赋值给SessionParams
            PackageInstaller.SessionParams params = new PackageInstaller.SessionParams(
                    PackageInstaller.SessionParams.MODE_FULL_INSTALL);
            ...

            try {
                // 注册InstallEventReceiver，并在launchFinishBasedOnResult会接收到安装事件的回调
                mInstallId = InstallEventReceiver
                        .addObserver(this, EventResultPersister.GENERATE_NEW_ID,
                                this::launchFinishBasedOnResult);
            } catch (EventResultPersister.OutOfIdsException e) {
                launchFailure(PackageManager.INSTALL_FAILED_INTERNAL_ERROR, null);
            }

            try {
                // createSession 内部通过IPackageInstaller与PackageInstallerService进行进程间通信，
                // 最终调用的是PackageInstallerService的createSession方法来创建并返回mSessionId
                mSessionId = getPackageManager().getPackageInstaller().createSession(params);
            } catch (IOException e) {
                launchFailure(PackageManager.INSTALL_FAILED_INTERNAL_ERROR, null);
            }
        }
            ...
    }
}
```

* 最终都会注册一个观察者 InstallEventReceiver，并在 launchFinishBasedOnResult 会接收到安装事件的回调，其中 InstallEventReceiver 继承自 BroadcastReceiver，用于接收安装事件并回调给 EventResultPersister
* createSession 内部通过 IPackageInstaller 与 PackageInstallerService 进行进程间通信，最终调用的是 PackageInstallerService的createSession 方法来创建并返回 mSessionId
* 接下来在 onResume 方法创建 InstallingAsyncTask 用来执行 APK 的安装，接着查看 onResume 方法

```
protected void onResume() {
    super.onResume();
    if (mInstallingTask == null) {
        PackageInstaller installer = getPackageManager().getPackageInstaller();
        // 根据mSessionId 获取SessionInfo, 代表安装会话的详细信息
        PackageInstaller.SessionInfo sessionInfo = installer.getSessionInfo(mSessionId);
        if (sessionInfo != null && !sessionInfo.isActive()) {
            mInstallingTask = new InstallingAsyncTask();
            mInstallingTask.execute();
        } else {
            // 安装完成后会收到广播
            mCancelButton.setEnabled(false);
            setFinishOnTouchOutside(false);
        }
    }
}
```

得到 SessionInfo 创建并创建 InstallingAsyncTask，InstallingAsyncTask 的 doInBackground 方法设置安装进度条，并将 APK 信息写入 PackageInstaller.Session，写入完成之后，在 InstallingAsyncTask 的 onPostExecute 进行成功与失败的处理，接着查看 onPostExecute 方法

```
protected void onPostExecute(PackageInstaller.Session session) {
    if (session != null) {
        Intent broadcastIntent = new Intent(BROADCAST_ACTION);
        broadcastIntent.setFlags(Intent.FLAG_RECEIVER_FOREGROUND);
        broadcastIntent.setPackage(getPackageName());
        broadcastIntent.putExtra(EventResultPersister.EXTRA_ID, mInstallId);

        PendingIntent pendingIntent = PendingIntent.getBroadcast(
                InstallInstalling.this,
                mInstallId,
                broadcastIntent,
                PendingIntent.FLAG_UPDATE_CURRENT);

        session.commit(pendingIntent.getIntentSender());
        mCancelButton.setEnabled(false);
        setFinishOnTouchOutside(false);
    } else {
        getPackageManager().getPackageInstaller().abandonSession(mSessionId);

        if (!isCancelled()) {
            launchFailure(PackageManager.INSTALL_FAILED_INVALID_APK, null);
        }
    }
}
```

创建了 broadcastIntent，并通过 PackageInstaller.Session 的 commit 方法发送出去，通过 broadcastIntent 构造方法指定的 Intent 的 Action 为 BROADCAST_ACTION，而 BROADCAST_ACTION 是一个常量值

```
 private static final String BROADCAST_ACTION =
            "com.android.packageinstaller.ACTION_INSTALL_COMMIT";
```

回到 InstallInstalling.OnCreate 方法，在 OnCreate 方法注册 InstallEventReceiver，而 InstallEventReceiver 继承自 BroadcastReceiver，而使用 BroadcastReceiver 需要在 AndroidManifest.xml注册，接着查看 AndroidManifest.xml：
**/frameworks/base/packages/PackageInstaller/AndroidManifest.xml**

```
<receiver android:name=".InstallEventReceiver"
        android:permission="android.permission.INSTALL_PACKAGES"
        android:exported="true">
    <intent-filter android:priority="1">
        <action android:name="com.android.packageinstaller.ACTION_INSTALL_COMMIT" />
    </intent-filter>
</receiver>
```

安装结束之后，会在观察者 InstallEventReceiver 注册的回调方法 launchFinishBasedOnResult 处理安装事件的结果，接着查看 launchFinishBasedOnResult

```
private void launchFinishBasedOnResult(int statusCode, int legacyStatus, String statusMessage) {
    if (statusCode == PackageInstaller.STATUS_SUCCESS) {
        launchSuccess();
    } else {
        launchFailure(legacyStatus, statusMessage);
    }
}

private void launchSuccess() {
    Intent successIntent = new Intent(getIntent());
    successIntent.setClass(this, InstallSuccess.class);
    successIntent.addFlags(Intent.FLAG_ACTIVITY_FORWARD_RESULT);
    startActivity(successIntent);
    finish();
}
    
private void launchFailure(int legacyStatus, String statusMessage) {
    Intent failureIntent = new Intent(getIntent());
    failureIntent.setClass(this, InstallFailed.class);
    failureIntent.addFlags(Intent.FLAG_ACTIVITY_FORWARD_RESULT);
    failureIntent.putExtra(PackageInstaller.EXTRA_LEGACY_STATUS, legacyStatus);
    failureIntent.putExtra(PackageInstaller.EXTRA_STATUS_MESSAGE, statusMessage);
    startActivity(failureIntent);
    finish();
}
```

安装成功和失败，都会启动一个新的 Activity（InstallSuccess、InstallFailed）将结果展示给用户，然后 finish 掉 InstallInstalling<br/>

## 4. 总结

总结一下 PackageInstaller 安装APK的过程：

1. 根据根据 Uri 的 Scheme 找到入口 InstallStart
2. InstallStart 根据 Uri 的 Scheme 协议不同做不同的处理
3. 都会调用 PackageInstallerActivity, 然后分别对package协议和 file 协议的 Uri 进行处理
4. PackageInstallerActivity 检查未知安装源限制,如果安装源限制弹出提示 Dialog
5. 点击 OK 按钮确认安装后，会调用 startInstall 开始安装工作
6. 如果用户允许安装，然后跳转到 InstallInstalling，进行 APK 的安装工作
7. 在 InstallInstalling 中，向包管理器发送包的信息，然后注册一个观察者 InstallEventReceiver，并接受安装成功和失败的回调

## 5. 关于 packages.xml

在 Andorid 系统目录 “/data/system” 下保存很多系统文件，主要介绍 packages.xml 文件<br/>

* packages.xml：记录了系统中所有安装的应用信息，包括基本信息、签名和权限、APK 文件的路径、native 库的存储路径

系统启动的时候会通过 PackageManagerServcie 读取这个文件加载系统中所有安装的应用，这个文件在开发中也是非常有帮助的，不同厂商会对  Android 源码有不同的修改，如果我们需要分析系统 App 的源码，就通过这个 packages.xml 找到目标 APK，dump 出来分析源码<br/>

以下是 packages.xml 文件部分内容

```
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<packages>
    <version sdkVersion="27" databaseVersion="3" fingerprint="Meizu/meizu_M1822_CN/M1822:8.1.0/OPM1.171019.026/1539943691:user/release-keys" />
    <version volumeUuid="primary_physical" sdkVersion="27" databaseVersion="27" fingerprint="Meizu/meizu_M1822_CN/M1822:8.1.0/OPM1.171019.026/1539943691:user/release-keys" />
    <meizu_version meizu_fingerprint="8.1.0-1541573178_stable" />
    <permission-trees />
    <permissions>
        <item name="com.meizu.voiceassistant.push.permission.MESSAGE" package="com.meizu.voiceassistant" protection="2" />
        <item name="com.meizu.safe.alphame.permission.DATA" package="com.meizu.safe" protection="18" />
        <item name="android.permission.REAL_GET_TASKS" package="android" protection="18" />
        
        ......

        <item name="android.permission.MODIFY_PHONE_STATE" granted="true" flags="0" />
        <item name="com.android.launcher.permission.INSTALL_SHORTCUT" granted="true" flags="0" />
        <item name="android.permission.WAKE_LOCK" granted="true" flags="0" />
        </perms>
        <proper-signing-keyset identifier="1" />
    </package>
    
    <package name="com.android.providers.telephony" codePath="/system/priv-app/TelephonyProvider" nativeLibraryPath="/system/priv-app/TelephonyProvider/lib" primaryCpuAbi="arm64-v8a" publicFlags="1007402501" privateFlags="8" ft="11e8dc5d800" it="11e8dc5d800" ut="11e8dc5d800" version="27" sharedUserId="1001" isOrphaned="true" forceFull="true">
        <sigs count="1">
            <cert index="0" />
        </sigs>
        <perms>
            <item name="android.permission.SEND_RECEIVE_STK_INTENT" granted="true" flags="0" />
            <item name="android.permission.BIND_INCALL_SERVICE" granted="true" flags="0" />
            
            ......

            <item name="android.permission.UPDATE_APP_OPS_STATS" granted="true" flags="0" />
        </perms>
        <proper-signing-keyset identifier="1" />
    </package>
```

##### 5.1. package 表示包信息

* name 表示应用的包名
* codePath 表示的是 APK 文件的路径
* nativeLibraryPath 表示应用的 native 库的存储路径
* it 表示应用安装的时间
* ut 表示应用最后一次修改的时间
* version 表示应用的版本号
* userId 表示所属于的 id

##### 5.2. sign 表示应用的签名

* count 表示标签中包含有多少个证书
* cert 表示具体的证书的值

##### 5.3. perms 表示应用声明使用的权限，每一个子标签代表一项权限

## 6. 安利一个在线查看 Android 源码网站

[aospxref](http://aospxref.com/) 是 [weishu](https://zhuanlan.zhihu.com/weishu) 大神搭建一个在线查看在线查看 Android源码网站, 访问速度非常快<br/>

在这之前我常用的在线查看 Android 源码的网站 [androidxref](http://androidxref.com/)，访问速度不仅慢，而且更新也不及时，现在 Android 10 发布了，这个网站到现在提供的最新的代码还是 Andorid 9 <br/>

[aospxref](http://aospxref.com/) 提供了与 [androidxref](http://androidxref.com/) 完全一样的源码浏览和交叉索引功能；除此之外，它还有一些别的优点：

* 跟随 Android 版本更新，你永远可以看到最新的源代码。
* 服务器在阿里云，国内访问速度贼快。
* opengrok 版本较高，查阅代码时会有自动提示。
* 对页面做过部分优化，使用更便捷；比如可以在任意界面跳转到首页。<br/>

## 参考

* [Android包管理总结](https://cloud.tencent.com/developer/article/1199502)
* [安利一个看 Android 源代码的网站](https://zhuanlan.zhihu.com/p/90782268)
* [Android 权限](https://source.android.google.cn/devices/tech/config?hl=zh-cn)
* [Android包管理机制（一）PackageInstaller的初始化](https://mp.weixin.qq.com/s?__biz=MzAxMTg2MjA2OA==&mid=2649842440&idx=1&sn=a24633e9e82a74c6d33440ecddba1849&chksm=83bf6a53b4c8e345195a212b6e66abbc862ed40aa087730d24233d81e2cf9b22cdf81a7a14da&scene=21#wechat_redirect)

## 结语

致力于分享一系列 Android 系统源码、逆向分析、算法、翻译、Jetpack  源码相关的文章，如果你同我一样喜欢研究 Android 源码，可以关注我，如果你喜欢这篇文章欢迎 star，一起来学习，期待与你一起成长

**算法**

由于 LeetCode 的题库庞大，每个分类都能筛选出数百道题，由于每个人的精力有限，不可能刷完所有题目，因此我按照经典类型题目去分类、和题目的难易程度去排序

* 数据结构： 数组、栈、队列、字符串、链表、树……
* 算法： 查找算法、搜索算法、位运算、排序、数学、……

每道题目都会用 Java 和 kotlin 去实现，并且每道题目都有解题思路，如果你同我一样喜欢算法、LeetCode，可以关注我 GitHub 上的 LeetCode 题解：[Leetcode-Solutions-with-Java-And-Kotlin](https://github.com/hi-dhl/Leetcode-Solutions-with-Java-And-Kotlin)，一起来学习，期待与你一起成长

