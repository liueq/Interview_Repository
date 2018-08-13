# Hook Binder

Hook Binder  的目的是为了让插件也能够无缝的使用系统服务。

目前所理解到的 Hook Binder 实际作用是篡改Binder 调用。比如说在 Hook Binder 后，虽然插件Activity 无法进行跨进程通信，但是可以在 Hook点，使用 HolderActivity 代为执行？然后将结果返回给 PluginActivity。

## 尝试通过 Hook Binder，篡改 ClipboardService 的返回值

首先确认 Hook 点：

* IClipboard$Stub // ClipboardService 业务逻辑执行的地方，在此处加入我们自己的代码
* IClipboard$Proxy // Client 所获取到的 IBinder 对象持有者，本来应该是判断非本进程，从而进行 IPC 通信。在 Hook 之后，认为是本进程，直接返回 Stub 对象，而此 Stub 就是我们第一步中 Hook的对象。
* ServiceManager // 由于 Proxy 是保存在 ServiceManager 中，需要获取 `Map<String, IBinder> sCache`，将其替换为第2步中 Hook 的 Proxy。

以上三步完成后，就能够在不破坏系统行为的前提下，在 Binder 调用中插入我记得代码。

**Hook Stub 对象**

```java
public class BinderHookHandler implements InvocationHandler{

    Object base;

    /**
     * 这里传入的 IBinder 被视为是 BBinder，即本进程内通信
     * 所以下边的调用方法，也是直接调用。
     * @param binder
     * @param stub
     */
    public BinderHookHandler(IBinder binder, Class<?> stub) {

        try {
            Method method = stub.getMethod("asInterface", IBinder.class);
            method.setAccessible(true);
            base = method.invoke(stub, binder); //asInterface 得到的是 BBinder
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 当 proxy 是 BBinder 即 Stub 的时候，直接进行业务逻辑
        if ("getPrimaryClip".equals(method.getName())) {
            return ClipData.newPlainText(null, "You are hooked");
        }

        if ("hasPrimaryClip".equals(method.getName())) {
            return true;
        }

        return method.invoke(base, args);
    }

```

**Hook Proxy 对象**

```java
public class BinderProxyHookHandler implements InvocationHandler{
    IBinder base;
    Class<?> stub;
    Class<?> iinterface;
    /**
     * 构造方法传入的 IBinder 会被认为是 BpBinder，那么对应的就是 Stub.Proxy
     * @param binder
     */
    public BinderProxyHookHandler(IBinder binder) {
        base = binder;
        try {
            stub = Class.forName("android.content.IClipboard$Stub"); //Stub 名称就是 IInterface$Stub 这样的格式
            iinterface = Class.forName("android.content.IClipboard"); //IInterface 由 aidl 生成，名称就是 IClipboard
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if ("queryLocalInterface".equals(method.getName())) {
            //在 client 调用 queryLocalInterface，得到的应该是一个 Proxy 对象
            // 这里使用 Hook，返回我们自己的 Proxy
            // 注意实现的接口有3个，分别是 IBinder, IInterface，IClipboard，虽然3实现了2，但是仍然要写明
            return Proxy.newProxyInstance(proxy.getClass().getClassLoader(),
                    new Class[]{IBinder.class, IInterface.class, this.iinterface},
                    new BinderHookHandler(base, stub));
        }
        return method.invoke(base, args);
    }
}
```

**Hook ServiceManger 中保存的 IBinder**

```java
try {
    // 首先拿到 ServiceManager 的静态方法 getService，获得 IBinder 对象
    Class<?> clazz = Class.forName("android.os.ServiceManager");
    Method method = clazz.getDeclaredMethod("getService", String.class);
    method.setAccessible(true);
    IBinder rawBinder = (IBinder) method.invoke(null, "clipboard");
    // 创建 IBinder 的代理，进行 hook
    // 这个代理的目的是篡改 queryLocalInterface 方法的返回结果
    // 本来应该返回 Proxy，然后进行 IPC 通信
    // 在篡改之后，返回我们自己的 BinderProxyHookHandler
    // 这里的 queryLocalInterface 总是返回了我们自己的 Stub 对象
    // 意味着系统认为是本进程内的调用，从而直接执行 BinderHookHandler 中的方法
    IBinder hookBinder = (IBinder) Proxy.newProxyInstance(rawBinder.getClass().getClassLoader(), new Class[]{
            IBinder.class
    }, new BinderProxyHookHandler(rawBinder));
    Field cache = clazz.getDeclaredField("sCache");
    cache.setAccessible(true);
    Map<String, IBinder> map = (Map) cache.get(null);
    map.put("clipboard", hookBinder);
} catch (ClassNotFoundException e) {
    e.printStackTrace();
} catch (NoSuchMethodException e) {
    e.printStackTrace();
} catch (IllegalAccessException e) {
    e.printStackTrace();
} catch (InvocationTargetException e) {
    e.printStackTrace();
} catch (NoSuchFieldException e) {
    e.printStackTrace();
}
```







