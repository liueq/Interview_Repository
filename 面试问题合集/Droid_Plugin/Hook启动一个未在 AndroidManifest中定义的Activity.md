# Hook启动一个未在 AndroidManifest中定义的Activity

采用 StubActivity 的方式，在 AndroidManifest 中预制一个 Activity。当启动某个未声明的 Activity 时，在进入 AMS 之前，先将其替换为 StubActivity，然后在AMS处理完毕，回调到 App 时，再替换回来。

**进入 AMS 之前的 Hook 点**

mInstrumentation.execStartActivity()，通过使用 Proxy 的方式，在此方法中，将目标 Activity 替换为合法的 StubActivity.

> InstrumentationHandler 代理

```java
public class InstrumentationHandler extends Instrumentation {

    private static final String TAG = "InstrumentationHandler";

    private Instrumentation mRaw;

    public InstrumentationHandler(Instrumentation instrumentation) {
        mRaw = instrumentation;
    }

    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {

        // 设置合法的 Intent
        intent.setComponent(new ComponentName("com.weishu.intercept_activity.app", "com.liueq.test.StubActivity"));
        Log.d(TAG, "invoke: 成功替换掉了 Intent");
        ActivityResult result = null;
        try {
            Method method = Instrumentation.class.getDeclaredMethod("execStartActivity",
                    Context.class, IBinder.class, IBinder.class, Activity.class, Intent.class, int.class,
                    Bundle.class);
            method.setAccessible(true);
            result = (ActivityResult) method.invoke(mRaw, who, contextThread, token, target, intent, requestCode, options);
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
        return result;
    }

}
```

然后在 Activity.onCreate 中进行Hook 操作，注意必须要是 mInstrumentation 已经初始化完成才行，否则无法获得。

> MainActivity

```java
    private void hook() throws NoSuchFieldException, IllegalAccessException {
        // 反射获取 mInstrumentation
        Field mInstrumentation = Activity.class.getDeclaredField("mInstrumentation");
        mInstrumentation.setAccessible(true);
        Instrumentation raw = (Instrumentation) mInstrumentation.get(this);
        Log.d(TAG, "hook: raw instrumentation = " + raw.toString());
        // 设置代理
        InstrumentationHandler handler = new InstrumentationHandler(raw);
        mInstrumentation.set(this, handler);
        Log.d(TAG, "hook: proxy instrumentation = " + mInstrumentation.get(this).toString());
    }
```

至此，在启动一个未声明的 Activity时，会被替换成启动 StubActivity。

**从 AMS 返回之后的 Hook 点**

在完成了第一步 Hook后，只能够避免报错，但是未达到启动 TargetActivity 的目的。

从源代码分析可知，AMS 的调用来自于 ApplicationThread 这个 Binder。然后实际调用的是 ActivityThread 中的handleLaunchActivity 方法，由H这个handler 处理。

由于 Handler 的处理机制可知，需要一个 mCallback 即可拦截并篡改 Message 的信息。我们的目的是篡改 Intent 中的 ComponentName 对象。

> 自定义的 mCallback 

```java
Handler.Callback proxyCallback = new Handler.Callback() {
    @Override
    public boolean handleMessage(Message msg) {
        if (100 == msg.what) {
            Log.d(TAG, "handleMessage: H.LAUNCH_ACTIVITY 已经被 Hook。");
            try {
                Class<?> activityClientRecord = Class.forName("android.app.ActivityThread$ActivityClientRecord");
                Object record = msg.obj;
                Field intent = activityClientRecord.getDeclaredField("intent");
                intent.setAccessible(true);
                Intent rawIntent = (Intent) intent.get(record);
                rawIntent.setComponent(new ComponentName("com.weishu.intercept_activity.app", "com.liueq.test.TargetActivity"));
                intent.set(record, rawIntent);
                msg.obj = record;
            } catch (ClassNotFoundException e) {
                e.printStackTrace();
            } catch (NoSuchFieldException e) {
                e.printStackTrace();
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            }
        }
        return false;
    }
};
```

然后是 Hook 的操作，需要对 ActivityThread 中的 mH 设置上该 callback。

> MainActivity

```java
private void hook2() throws Exception{
    Field mainThread = Activity.class.getDeclaredField("mMainThread");
    mainThread.setAccessible(true);
    Object rawMainThread = mainThread.get(this);
    Class<?> activityThread = Class.forName("android.app.ActivityThread");
    Field h = activityThread.getDeclaredField("mH");
    h.setAccessible(true);
    Handler rawHandler = (Handler) h.get(rawMainThread);
    Field callback = Handler.class.getDeclaredField("mCallback");
    callback.setAccessible(true);
    callback.set(rawHandler, proxyCallback);
}
```

至此，可欺骗 AMS 启动未在 AndroidManifest 中声明的 Activity。并且此方法所启动的 Activity 是具有生命周期的。因为 AMS 对于 Activity 的识别是通过 Token。在这里虽然替换了 Activity，但是在 AMS 看来仍然是操作的 StubActivity。

在 ActivityThread 中使用 mActivity 来保存了所有的 Activity。通过 performLaunchActivity 方法，将 ActivityClientRecord 中保存的 Activiy 放入，然后在生命周期回调中，也是通过 ActivityClientRecord 来在 mActivity 中查找。因此上述篡改方法，能够将 TargetActivity 放入 mActivity，之后的生命周期回调自然也是取到的它。

