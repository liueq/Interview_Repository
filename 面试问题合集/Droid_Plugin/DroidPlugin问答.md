# Droid Plugin 问答

**Q：Droid Plugin 的优缺点？**

优点：

* 插件 APK 无需修改，可以单独运行，也可以作为插件运行。
* 插件中的四大组件无需在 Host 中注册。
* Host 继承 DroidPlugin 只需要极少代码。
* 插件之间，Host 之间代码级别隔离，只能通过系统提供的调用进行通信。
* 支持所有的系统API

缺点：

* 需要预先注册所有插件中可能会申请的权限。
* Manifest 中需要事先添加大量的占坑组件。
* 一些特殊 IntentFilter 组件无法被系统或者其他 App 调用（因为真正系统认知的 Manifest 只是占坑的，并不知道插件中具体是什么）。
* 缺乏对 native 层的 hook。

**Q：Droid Plugin 实现的原理。**

核心思想是通过 hook 系统 API，对插件中组件的操作在进入 frameworks 之前，用Host中的占坑组件代替，在 frameworks 回调时，替换对插件中组件的操作。

Hook 主要是通过反射，用代理的方式来进行请求或者回调的篡改。

Activity 主要是代理，而 Broadcast 则是用动态广播来代理所有的静态，动态广播。Service 则是用一个 Service 来管理多个。

**Q：简述启动插件 Activity 的过程。**

主要有两点，其一，加载外部 apk 中的 Activity 类和资源；其二，Hook AMS 从而能够通过 startActivity 调用未在 manifest 中定义的 Activity。

加载外部APK：

1. 使用 PackageParser 解析 apk 文件，得到 ApplicationInfo。
2. 反射创建出 LoadedApk，其中包含了插件 APK 的所有信息。
3. 创建插件 apk 的 DexClassloader，并加入到 LoadedApk中。
4. 将 LoadedApk 加入到 ActivityThread 中的 mPackages field 中。因为在创建Activity时，会根据包名来查找对应的 LoadedApk 对象，然后用其中的 Classloader 来加载目标 Activity。
5. 至此，外部Activity class 理论上可以被创建了。

Hook AMS，绕过 manifest 中未注册的检测：

1. 将 mInstrumentation 替换为代理类，启动目标 Activity时，将 ComponentName 替换为一个 Host manifest 中定义的占坑 Activity。这样就可以通过系统的检测。
2. 当系统回调时，默认会调用 host 中占坑 Activity。此时应该创建一个 Handler.Callback 对象，替换到 ActivityThread.mH 这个 handler 中的 callback。AMS 对于 Activity 的回调都会经过该 handler。
3. 在创建 Activity 回调时，将占坑 Activity 又替换回目标 Activity。在已经将该 Activity class 加载且，mPackages 中已经保存了插件Apk 的信息时，就可以成功创建了。

**Q：Droid Plugin 如何加载插件APK的？**

通过 ClassLoader 来加载插件中的类，而资源文件是通过反射创建 AssetManager，从而实现加载插件中的资源。

对于 Apk 的解析是通过 PackageParser。

**Q：Binder Hook 的原理？**

当使用 Binder 进行时，由于 Binder 实现原理，当判断目标是非本进程，那么就会开始进行跨进程通信。Binder Hook 就是通过篡改 Binder 判断是否外部进程的方式（能否 queryLocalInterface）。从而让 Binder 认为该请求是一个本地的通信，那么就直接返回 Stub 供调用。