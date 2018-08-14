# Hook加载一个外部的Activity

这篇笔记主要是讲如何从一个外部的apk中加载Activity。根据实现方式的不同，主要由两种方式。

* 使用一个新的 ClassLoader 来加载。
* 将外部apk 的路径加入当前 ClassLoader 的 dex path 中。

## 方案一，使用新的 ClassLoader 来进行加载

本方案的解决思路是通过创建一个以外部apk为资源的 ClassLoader，然后hook Activity 的创建流程，使用该 ClassLoader 来创建一个Activity。主要分为以下步骤：

* 在启动Activity 的时候，进入 AMS 之前，先替换为 StubActivity。
* 在从 AMS 返回时，从ActivityThread.mH 这个 Handler 中，将 StubActivity 还原为我们要启动的 TargetActivity。
* 在 ActivityThread.mPackages 中添加 package to LoadedApk 的键值对，这里的 LoadedApk 需要解析外部 apk 生成，其中包括创建 ClassLoader。
* 由于创建过程中，会在 PMS 中查询一次 PackageInfo，所以需要 Hook PMS，在查询外部apk的包名时，返回一个 PackageInfo 对象。

至此，就能够启动一个外部 apk 中的 Activity了。但是由于没有加载资源文件，Activity 仍然无法显示。

### 1. 在进入 AMS 之前，先将 TargetActivity 替换为本包内的 StubActivity

首先创建 Instrumentation 的代理，覆盖 execStartActivity 方法，将 ComponentName 替换。

> InstrumentationProxy

```java
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
    Log.d(TAG, "execStartActivity: 成功 Hook");
    try {
        intent.setComponent(new ComponentName("com.weishu.upf.hook_classloader", "com.liueq.test.StubActivity"));
        Method method = Instrumentation.class.getDeclaredMethod("execStartActivity",
                Context.class, IBinder.class, IBinder.class, Activity.class, Intent.class,
                int.class, Bundle.class);
        method.setAccessible(true);
        return (ActivityResult) method.invoke(mRaw, who, contextThread, token, target, intent, requestCode, options);
    } catch (NoSuchMethodException e) {
        e.printStackTrace();
    } catch (IllegalAccessException e) {
        e.printStackTrace();
    } catch (InvocationTargetException e) {
        e.printStackTrace();
    }
    return null;
}
```

然后替换掉 Activity 中的 mInstrumentation。

> MainActivity.onCreate()

```java
private void hook1() throws NoSuchFieldException, IllegalAccessException {
    Field instrumentation = Activity.class.getDeclaredField("mInstrumentation");
    instrumentation.setAccessible(true);
    Instrumentation raw = (Instrumentation) instrumentation.get(this);
    instrumentation.set(this, mInstrumentationProxy = new InstrumentationProxy(raw));
}
```

### 2. 从 AMS 返回后，在 ActivityThread.mH 中拦截创建 Activity 的消息，将 StubActivity 还原为 TargetActivity

首先创建一个 Hanndler.Callback 对象，当 Message.what 为创建Activity时，替换 ComponentName。

> Callback

```java
private Handler.Callback mCallback = new Handler.Callback() {
    @Override
    public boolean handleMessage(Message msg) {
        if (100 == msg.what) {
            try {
                Class<?> activityClientRecord = Class.forName("android.app.ActivityThread$ActivityClientRecord");
                Object rawActivityClientRecord = msg.obj;
                Field intent = activityClientRecord.getDeclaredField("intent");
                intent.setAccessible(true);
                Intent rawIntent = (Intent) intent.get(rawActivityClientRecord);
                rawIntent.setComponent(new ComponentName("com.weishu.upf.ams_pms_hook.app", "com.weishu.upf.ams_pms_hook.app.MainActivity"));
                intent.set(rawActivityClientRecord, rawIntent);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        return false;
    }
};
```

使用反射往 ActivityThread.mH 这个 Handler 中设置新的 mCallback（本来为 null）。

> MainActivity

```java
private void hook2() throws NoSuchFieldException, IllegalAccessException, ClassNotFoundException {
    Field mainThread = Activity.class.getDeclaredField("mMainThread");
    mainThread.setAccessible(true);
    rawActivityThread = mainThread.get(this);
    Class<?> activityThread = Class.forName("android.app.ActivityThread");
    Field handler = activityThread.getDeclaredField("mH");
    handler.setAccessible(true);
    Handler rawHandler = (Handler) handler.get(rawActivityThread);
    Field mCallback = Handler.class.getDeclaredField("mCallback");
    mCallback.setAccessible(true);
    mCallback.set(rawHandler, this.mCallback);
}
```

### 3. 向 ActivityThread.mPackages 中添加新的映射

ActivityThread.mPackages 是一个类型为 `Map<String, WeakReference<LoadedApk>>` 的成员变量。在创建一个Activity 时候，会根据包名从中获取 LoadedApk，然后使用里边的 ClassLoader 来加载这个 Activity。

> 将 LoadedApk 与包名放入 mPackages 中

```java
private void hook3() throws ClassNotFoundException, NoSuchFieldException, IllegalAccessException, NoSuchMethodException, InvocationTargetException, InstantiationException {
    Class<?> loadedApk = Class.forName("android.app.LoadedApk");
    // 需要创建一个 LoadedApk
    Object hookedLoadedApk = getLoadedApk();
    Class<?> activityThread = Class.forName("android.app.ActivityThread");
    Field mPackages = activityThread.getDeclaredField("mPackages");
    mPackages.setAccessible(true);
    ArrayMap<String, WeakReference<?>> packages = (ArrayMap<String, WeakReference<?>>) mPackages.get(rawActivityThread);
    Field packageName = loadedApk.getDeclaredField("mPackageName");
    packageName.setAccessible(true);
    String pn = (String) packageName.get(hookedLoadedApk); // 注意，这里必须是外部包名
    packages.put(pn, new WeakReference<Object>(hookedLoadedApk));
    mPackages.set(rawActivityThread, packages);
}
```

LoadedApk 可以通过 ActivityThread.getPackageInfo() 创建，而该方法的参数有: ApplicationInfo, CompatibilityInfo。因此也需要先创建这两个对象。

> 生成 LoadedApk

```java
private Object getLoadedApk() throws ClassNotFoundException, NoSuchMethodException, InvocationTargetException, IllegalAccessException, InstantiationException, NoSuchFieldException {
    Class<?> activityThread = Class.forName("android.app.ActivityThread");
    Class<?> compatibilityInfo = Class.forName("android.content.res.CompatibilityInfo");
    Method getPackageInfo = activityThread.getDeclaredMethod("getPackageInfo", ApplicationInfo.class, compatibilityInfo, ClassLoader.class, boolean.class, boolean.class, boolean.class);
    getPackageInfo.setAccessible(true);
    
    // 需要先创建 ApplicationInfo 和 CompatibilityInfo
    Object rawLoadedApk = getPackageInfo.invoke(rawActivityThread, getHookApplicationInfo(), getCompatibilityInfo(), getHookClassLoader(), false, true, false);
    Class<?> loadedApk = Class.forName("android.app.LoadedApk");
    Field classLoader = loadedApk.getDeclaredField("mClassLoader");
    classLoader.setAccessible(true);
    classLoader.set(rawLoadedApk, mHookedClassLoader);
    return rawLoadedApk;
}
```

ApplicationInfo 需要使用 PackageParser 来解析 apk 文件，而CompabilityInfo 反射获取默认的值即可。

> 生成 ApplicationInfo

```java
private ApplicationInfo getHookApplicationInfo() throws ClassNotFoundException, NoSuchMethodException, IllegalAccessException, InstantiationException, InvocationTargetException {
    Class<?> packageParser = Class.forName("android.content.pm.PackageParser");
    Class<?> cPackage = Class.forName("android.content.pm.PackageParser$Package");
    Class<?> packageUserState = Class.forName("android.content.pm.PackageUserState");
  
  //需要使用 generateApplicationInfo 方法和 parsePackage 方法
    Method generateApplicationInfo = packageParser.getDeclaredMethod("generateApplicationInfo", cPackage, int.class, packageUserState);
    Method parsePackage = packageParser.getDeclaredMethod("parsePackage", File.class, int.class);
    parsePackage.setAccessible(true);
    generateApplicationInfo.setAccessible(true);
    Object thePackageParser = packageParser.newInstance();
    Object p = parsePackage.invoke(thePackageParser, mApkFile = getApkFile(), 0);
    ApplicationInfo aInfo = (ApplicationInfo) generateApplicationInfo.invoke(null, p, 0, packageUserState.newInstance()); //静态方法直接使用
    Log.d(TAG, "getHookApplicationInfo: 生成 ApplicationInfo时，其中的报名为??" + aInfo.packageName);
    return aInfo;
}


```

> 将外部 apk 移动到 内部存储中

```java
/**
 * 从 assets 中将 apk 放到 /data/data/packages/plugins/下
 * @return
 */
private File getApkFile() {
    AssetManager assetManager = getAssets();
    InputStream is = null;
    FileOutputStream fos = null;
    File out = null;
    try {
        is = assetManager.open("ams-pms-hook.apk");
        Resources resources = getResources();
        out = getFileStreamPath("ams-pms-hook.apk");
        fos = new FileOutputStream(out);
        byte[] buffer = new byte[1024];
        int len = 0;
        while ((len = is.read(buffer)) != -1) {
            fos.write(buffer, 0, len);
        }
    } catch (IOException e) {
        e.printStackTrace();
    }finally {
        try {
            fos.flush();
            fos.close();
            is.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    return out;
}
```

> 反射创建 CompatibilityInfo

```java
/**
 * 获取一个默认的 CompatibilityInfo 对象
 */
private Object getCompatibilityInfo() throws ClassNotFoundException, NoSuchFieldException, IllegalAccessException {
    Class<?> compatibilityInfo = Class.forName("android.content.res.CompatibilityInfo");
    Field defaultInfo = compatibilityInfo.getDeclaredField("DEFAULT_COMPATIBILITY_INFO");
    defaultInfo.setAccessible(true);
    return defaultInfo.get(null);
}
```

至此就可以创建 LoadedApk了，然后需要使用自定义的 ClassLoader 来替换 LoadeApk 中的 mClassLoader。需要注意的是，LoadedApk 的构造方法可以传入一个 ClassLoader，该变量会在 LoadedApk 中保存为 mBaseClassLoader。而ActivityThread 获取 LoadedApk 的ClassLoader 是通过 getClassLoder() 方法，该方法返回的是 mClassLoader。所以正确的做法应该是反射替换掉 mClassLoader。

> 创建ClassLoader

```java
private ClassLoader getHookClassLoader() {
    File plugin = getFileStreamPath("plugin");
    plugin.mkdir();
    File opt_dex = new File(plugin, "opt_dex");
    File lib = new File(plugin, "lib");
    opt_dex.mkdir();
    lib.mkdir();
  
  //注意参数应该是需要读取 apk 的路径以及解释出 dex 文件的路径。
    mHookedClassLoader = new DexClassLoader(mApkFile.getAbsolutePath(), opt_dex.getAbsolutePath(), lib.getAbsolutePath(), ClassLoader.getSystemClassLoader());;
    Log.d(TAG, "getHookClassLoader: 自己创建的 classloader  " + mHookedClassLoader.toString());
    return mHookedClassLoader;
}
```

至此，在 ActivityThread.performLaunchActivity 中就可以生成一个外部 Activity 的对象了。

### 4. 处理创建 Activity 中对于 PMS 的一次查询

在上一步完成后，后续会调用 LoadedApk.makeApplication() 方法来创建 Application 对象。该过程中，会调用initializeJavaContextClassLoader() 方法，此方法用该包名去查询 PackageInfo。如果不做处理，这里的包名是一个外部包名，系统认为未安装。

> 创建 PackageManagerProxy，处理一下 getPackageInfo 方法

```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    Log.d(TAG, "invoke: PMS 被 hook 了");
    if ("getPackageInfo".equals(method.getName())) {
        //Hook
        if (args[0] instanceof String) {
            String pkgName = (String) args[0];
            if ("com.weishu.upf.ams_pms_hook.app".equals(pkgName)) {
                Log.d(TAG, "invoke: 使用了自己的 PackageInfo");
                return new PackageInfo();
            }
        }
    }
    return method.invoke(raw, args);
}
```

> MainActivity.onCreate() 替换 sPackageManager

```java
/**
 * hook PMS 中的 getPackageInfo 方法， 因为在 LoadedApk 中的 initializeJavaContextClassLoader 方法会查询一次 PMS
 * 这里必须要返回一个 PackageInfo
 */
private void hook5() throws ClassNotFoundException, NoSuchFieldException, IllegalAccessException {
    Class<?> stub = Class.forName("android.content.pm.IPackageManager"); //由于只是hook proxy，所以这里就是IPackageManager
    Class<?> activityThread = Class.forName("android.app.ActivityThread");
    Field sPackageManager = activityThread.getDeclaredField("sPackageManager");
    sPackageManager.setAccessible(true);
    Object iPackageManager = sPackageManager.get(null);
    Object proxy = Proxy.newProxyInstance(iPackageManager.getClass().getClassLoader(), new Class[]{
        stub
    }, new PackageManagerProxy(iPackageManager));
    sPackageManager.set(null, proxy);
}
```

至此，使用单独 ClassLoader 的方式来加载 Activity 就结束了。

## 方案二，在默认 ClassLoader 中追加 dex path，使得其可以加载外部的 apk

在一个 App中，默认只有一个 ClassLoader。所以任何地方获取的 ClassLoader 都是同一个。

在Android 系统中，ClassLoader 的继承关系如下：

PathClassLoader -> BaseDexClassLoader -> ClassLoader

通过观察 findClass 方法可知，ClassLoader 在加载一个新的 Class 时，查询的路径是定义在了 BaseDexClassLoader.pathList 中。

而 DexPathList 通过 Element[] 这样一个数组来保存每一个路径。

因此，本方案所要做的就是往 Elemnt[] 中添加一个新的路径。

> MainActivity.onCreate()

```java
private void hook6() throws Exception {
    ClassLoader rawClassLoader =  getClassLoader();
    Field dexPathList = BaseDexClassLoader.class.getDeclaredField("pathList");
    dexPathList.setAccessible(true);
    Object rawDexPathList = dexPathList.get(rawClassLoader);
    Class<?> DexPathList = Class.forName("dalvik.system.DexPathList");
    Field dexElements = DexPathList.getDeclaredField("dexElements");
    dexElements.setAccessible(true);
    // 注意这里的 rawDexElements 是数组
    Object[] rawDexElemnets = (Object[]) dexElements.get(rawDexPathList);
    // 反射生成数组对象的方法
    Class<?> Element = Class.forName("dalvik.system.DexPathList$Element");
    Object[] expandDexElements = (Object[]) Array.newInstance(Element, rawDexElemnets.length + 1);
    for(int i = 0 ;i < rawDexElemnets.length; i++) {
        expandDexElements[i] = rawDexElemnets[i];
    }
    Constructor<?> constructor = Element.getConstructor(File.class, boolean.class, File.class, DexFile.class);
    constructor.setAccessible(true);
    
    // 创建一个新的 Element，注意 DexFile 的构造方法参数
    Object newElement = constructor.newInstance(getApkFile(), false, getApkFile(),
            DexFile.loadDex(getApkFile().getAbsolutePath(), getApkFile().getParentFile().getAbsolutePath() + "ams.dex", 0));
    expandDexElements[expandDexElements.length - 1] = newElement;
    dexElements.set(rawDexPathList, expandDexElements);
}
```

其余 AMS 之前和之后的 hook 和方案一相同，需要替换和换回 StubActivity。

至此，方案二可以加载一个外部的 Activity了。

