# Hook AMS & PMS

**ActivityManagerService的重要性**

1. startActivity 最终调用了 AMS 中的 startActivity 方法，实现了 Activity 的启动。而 Activity 的生命周期回调也在 AMS 中完成。
2. startService 和 bindService 最终也会调用到 AMS 的 startService 和 bindService。
3. 动态广播的注册和接收在 AMS 中完成（静态广播在 PMS 中完成）。
4. getContentResolver 最终从 AMS 的 getContentProvider 获取到 ContentProvider。

**PackageManagerServcie的重要性**

权限校验（checkPermission, checkUidPermission），Apk meta 信息的获取（getApplicationInfo），四大组件信息获取（query系列方法）等。

**AMS Hook 点寻找**

从 startActivity 开始，最终会调用到 ActivityManagerNative 的 startActivity 方法。

AMN 是 server 端的服务，原本应该是通过 Binder 通信进行调用。然后由于 AMS 作为一个频繁使用的系统服务，系统将 AMN 作为单例进行了保存。

所以并不需要 Hook Binder，只需要替换掉 AMN 所保存的单例即可。

**实现代码**

> IActivityManager Proxy 的定义

```java
public class HookAmsHandler implements InvocationHandler{

    String TAG = "HookAmsHandler";

    Object mBase;

    public HookAmsHandler(Object base) {
        mBase = base;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      //可以 hook 任何方法，这里只是简单的打印日志
        if ("startActivity".equals(method.getName())) {
            Log.d(TAG, "invoke: AMS 已经被 Hook 了");
        }
        return method.invoke(mBase, args);
    }
}
```

> Hook 设置

```java
private void hookAms() {
    try {
        Class<?> iActivityManager = Class.forName("android.app.IActivityManager");
        
        // 获取 ActivityManagerNative 中 gDefault 这个静态变量
        Class<?> activityManagerNative = Class.forName("android.app.ActivityManagerNative");
        Field gDefault = activityManagerNative.getDeclaredField("gDefault");
        gDefault.setAccessible(true);
        Object object = gDefault.get(null);
        // 由于 gDefault 是 Singleton 类型，所以这里拿到 Singleton 类中的 mInstance field
        Class<?> singleton = Class.forName("android.util.Singleton");
        Field mInstance = singleton.getDeclaredField("mInstance");
        mInstance.setAccessible(true);
        // 获取原始的 IActivityManager，注意 invoke object，即 gDefault 。
        Object rawIActivityManager = mInstance.get(object);
        // 生成 Proxy
        Object proxy = Proxy.newProxyInstance(rawIActivityManager.getClass().getClassLoader(), new Class[]{
                iActivityManager, IInterface.class
        }, new HookAmsHandler(rawIActivityManager));
        // 替换 gDefault 的值
        mInstance.set(object, proxy);
    } catch (ClassNotFoundException e) {
        e.printStackTrace();
    } catch (IllegalAccessException e) {
        e.printStackTrace();
    } catch (NoSuchFieldException e) {
        e.printStackTrace();
    }
}
```

**PMS Hook 点寻找**

通过分析 PackageManager，可以得知 IPackageManager 来自于 

```javascript
IPackageManager pm = ActivityThread.getPackageManager()
mPackageManager = new ApplicationPackageManager(this, pm); //封装了 IPackageManager
```

所以需要 Hook 的地方有两处：

* ActivityThread 中的 sPackageManager 
* ApplicationPackageManager 中的 mPM

**实现代码**

> IPackageManager Proxy 类

```java
public class HookPmsHandler implements InvocationHandler{
    private String TAG = "HookPmsHandler";
    Object mBase;
    public HookPmsHandler(Object base) {
        mBase = base;
    }
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      // 只是简单的打印日志
        if ("getInstalledApplications".equals(method.getName())) {
            Log.d(TAG, "invoke: Pms 已经被 Hook 了");
        }
        return method.invoke(mBase, args);
    }
}
```

> Hook 设置

由于 IPackageManager 并不是一个静态变量或者单例， 所以需要 Hook 每一个对象。这里是 Hook 了两处。

```java
try {
    Class<?> cIPackageManager = Class.forName("android.content.pm.IPackageManager");
    Class<?> cActivityThread = Class.forName("android.app.ActivityThread");
    Class<?> cApplicationPackageManager = Class.forName("android.app.ApplicationPackageManager");
  
    // Hook ActivityThread 中的 IPackageManager
    Field sPackageManager = cActivityThread.getDeclaredField("sPackageManager");
    sPackageManager.setAccessible(true);
    sPackageManager.setAccessible(true);
    Object rawIPackageManager = sPackageManager.get(null);
    Object proxy = Proxy.newProxyInstance(rawIPackageManager.getClass().getClassLoader(), new Class[]{
            cIPackageManager, IInterface.class
    }, new HookPmsHandler(rawIPackageManager));
    sPackageManager.set(null, proxy);
  
    // Hook ContextImpl 中的 IPackageManager
    Class<?> cContextImpl = Class.forName("android.app.ContextImpl");
    Field mPackageManager = cContextImpl.getDeclaredField("mPackageManager");
    mPackageManager.setAccessible(true);
    Object packageManager = mPackageManager.get(context);
    Field mPM = cApplicationPackageManager.getDeclaredField("mPM");
    mPM.setAccessible(true);
    Object rawPm = mPM.get(packageManager);
    Object proxy2 = Proxy.newProxyInstance(rawPm.getClass().getClassLoader(), new Class[]{
            cIPackageManager, IInterface.class
    }, new HookPmsHandler(rawPm));
    mPM.set(packageManager, proxy2);
    mPackageManager.set(context, packageManager);
} catch (ClassNotFoundException e) {
    e.printStackTrace();
} catch (NoSuchFieldException e) {
    e.printStackTrace();
} catch (IllegalAccessException e) {
    e.printStackTrace();
}
```

