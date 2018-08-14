# Hook 机制原理

Hook 系统 API，在应用调用系统 API 的时候，实际上使用的使我们自己的 Proxy 类，为插件化实现作铺垫。

Hook 点的选择非常重要，一般是选择静态变量或者单例。如果是普通对象，要么无法标志，要么容易改变。

## Hook startActivity 方法

**首先分析 startActivity 的调用栈**

* startActivity()
* ContextImpl.startActivity()
* ActivityThread.getInstrumentation().execStartActivity()
* Instrumentation.execStartActivity()

由此可以得出需要代理的对象是 Instrumentation。

**Hook 时机**

在 attachBaseContext 时候，第一次得到 Context 对象，此时就可以开始。

```java
@Override
protected void attachBaseContext(Context newBase) {
    super.attachBaseContext(newBase);
    try {
      //首先反射得到 ActivityThread class
        Class<?> activityThread = Class.forName("android.app.ActivityThread");
      
      //找到 ActivityThread 的单例 sCurrentActivityThread
        Field sCurrentActivityThread = activityThread.getDeclaredField("sCurrentActivityThread");
        sCurrentActivityThread.setAccessible(true);
        Object currentActivityThread = sCurrentActivityThread.get(null);
      
      //得到 ActivityThread 中所保存的 mInstrumentation 对象，并替换成自己的
        Field instrumentation = (currentActivityThread).getClass().getDeclaredField("mInstrumentation");
        instrumentation.setAccessible(true);
        instrumentation.set(currentActivityThread, new MyInstrumentation((Instrumentation) instrumentation.get(currentActivityThread)));
    } catch (NoSuchFieldException e) {
        e.printStackTrace();
    } catch (IllegalAccessException e) {
        e.printStackTrace();
    } catch (ClassNotFoundException e) {
        e.printStackTrace();
    }
    context = newBase;
}
```

**Proxy 类的编写**

```java
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
  
  //自己的业务逻辑
    System.out.println("liueq : 是的，已经被 Hook。");

  //最后还是要调用系统的方法以实现功能  
  Method method = null;
    ActivityResult result = null;
    try {
        method = Instrumentation.class.getDeclaredMethod("execStartActivity", Context.class,
                IBinder.class, IBinder.class, Activity.class, Intent.class, int.class, Bundle.class);
        method.setAccessible(true);
        result = (ActivityResult) method.invoke(mBase, who, contextThread, token, target, intent, requestCode, options);
    } catch (NoSuchMethodException e) {
        e.printStackTrace();
    } catch (IllegalAccessException e) {
        e.printStackTrace();
    } catch (InvocationTargetException e) {
        e.printStackTrace();
    }
    return result;
}
```



