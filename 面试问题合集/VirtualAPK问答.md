# VirtualAPK 问答

**Q：简述 VirtualAPK 的架构。**

A：由以下几个重要模块组成：

* 核心组件：包括 PluginManager, LoadedPlugin 等。调度整个插件框架。
* Activity 相关组件：包括 VAInstrument，介于 Plugin 和 frameworks 之间，主要关注 Activity 的代理。
* Service 相关组件：包括 ActivityManagerService, LocalService, RemoteService 等，主要关注 Service 代理的实现。
* ContentProvider 相关组件：包括IContentProviderProxy，RemoteContentProvder 等，主要关注 ContentProvider 代理。
* Resource 相关组件：包括 ResourceManager，PackageParser 等，负责插件资源文件的管理。

VirtualAPK 代理了所有 plugin 与 frameworks 的交互。由于使用的 ClassLoader 和 Host 是同一个，所以 plugin 和 host 之间能够交互，这也是 VirtualAPK 的主要使用场景。

**Q：VirtualAPK 如何处理 plugin 的资源文件。**

A：在创建 LoadedPlugin 对象的时候，对于其中的资源，主要有两种处理方式。其一是与当前 host 资源合并；其二是作为 Resources 对象使用。

与 host 合并的情况下，只要按照以下几个步骤：

1. 在 lollipop 之前，需要创建一个新的 AssetManager 对象，反射调用 "addAssetPath" 先将 host 的资源加入。在 lollipop 及其之后可以从直接从 host context 中获取 AssetManager。
2. 将当前 plugin apk 加入 AssetManager。
3. 将所有的 plugin resource 加入 AssetManager。
4. 使用 AssetManager 创建 Resources 对象（大多数手机 rom 都需要适配，因为 Resources 对象自定义名称居多）。

**Q：ClassLoader 的创建过程。**

A：如果不需要合并 ClassLoader，那么直接创建一个新的 DexClassLoader 即可。

默认是需要合并 ClassLoader，按照下面步骤进行：
0. 仍然是创建 plugin 的 DexClassLoader。
1. 分别获取 host，plugin ClassLoader 中的 dexElements（BaseClassLoader.pathList.dexElements）。
2. 合并两个 array，让 host dexElements 排在 array 的前边。
3. 反射替换掉当前 host 的 ClassLoader 中的 dexElements。
4. 最后还需要合并一下 native library

**Q：VirtualAPK 对于 plugin 中 Activity 的代理流程。**

A：主要是通过代理 Instrumentation 类，在 framework 进行相关检查之前，将 plugin activity 替换为 stub activity，且在 framework 检查完毕后，回到 Instrumentation 时，将启动的目标 activity 又换回 plugin activity。

值得注意的一些细节：

1. 在 execStartActivity 阶段，intent 全部转换为了 explicit intent. 在使用 stub activity 替换时，将原本 plugin activity 信息存放到了 bundle 中。在挑选 plugin activity 时，根据启动参数不同，从 manifest 中组合出符合要求的 stub activity。
2. newActivity 阶段，第一步直接用 host classloader 加载该 activity。如果是 stub activity 的情况。由于所有的 stub activity 都只是定义在了 manifest 中，并没有真正的 class。所以必定会抛出异常。在处理异常时，从 bundle 获取 plugin activity 信息，然后使用 classloader 加载。使用异常处理的巧妙地方在于，如果是启动 host 中的 activity，那么不会影响其流程。
3. callActivityOnCreate 阶段，会给 plugin activity 注入 plugin resources。

**Q：VirutalAPK 对于 plugin 中 Service 的代理流程。**

A：

正常的 service 启动流程：
ContextImpl
	.startService
	.startServiceCommon
	ActivityManagerNative.getDefault().startService
通过 getDefault 得到的就是 IActivityManager 对象，已经被代理。

通过代理 startService方法，将 plugin service 封装到 bundle 中，启动代理service。
ActivityManagerProxy
	.startService
	.startDelegateServiceForTarget
	.wrapTargetIntent # 将plugin service封装到 bundle中，启动代理service
这里启动了 LocalService or RemoteService。

分析 LocalService：
LocalService 应该扮演的是Binder本地调用的情景。

onStartCommand 方法代理了 plugin service。EXTRA_TARGET 包含了 plugin service 的信息，EXTRA_COMMAND 包含了调用的 plugin service 具体方法。根据 EXTRA_COMMAND 的不同，通用的方式如下：
1. 首先判断 service 是否创建，未创建则创建，已创建则从 cache 中获取。
2. 调用 EXTRA_COMMAND 对应的方法。

bind 方法会涉及到一些 Binder 调用，由于这里都是本地所以没有太大问题。

分析 RemoteService:
RemoteService 继承了 LocalService。在 RemoteService 的情况下，plugin service 所在插件还未被加载，所以多了加载插件的步骤，其他逻辑和 LocalService 无异。

由于 Service 的生命周期比 Activity 简单，且触发 Service 生命周期方法也是通过手动调用，所以才能够用这样的代理方式。Service 被创建后，和 frameworks 就没有关系了。

**Q：VirtualAPK 对于 plugin 中 ContentProvider 的代理流程。**

A：

分析ContentProvider 的代理流程：

Activity
	.getContentResolver
ContextImpl
	.getContentResolver
得到 ApplicationContentResolver

ContentResolver
	.query
	.acquireUnstableProvider
ActivityThread
	.acquireProvider
	.acquireExistingProvider # 尝试从 mProviderMap 中获取。mProviderMap 是个 map 对象，保存 auth -> IContentProvider
ActivityManagerNative.getNative()
	.getContentProvider # 又从 ActivityManager 获取，这里 hook 了 IContentProvider
ActivityThread
	.installProvider() # 从 ContentProvider 获取 IContentProvider 查询接口

得到 IContentProvider，现在可以直接通过 IContentProvider 查询得到 Cursor.


VirtualApk 对于 ContentProvider 的处理：

IContentProviderProxy 代理掉了 IContentProvider，任何操作都会先对 Uri 进行包装。将 plugin content provider 作为参数，实际上先对 RemoteContentProvider 进行查询。

RemoteContentProvider 代理 ContentProvider。将查询分发到对应的 plugin content provider。如果该 plugin 未加载，先进行加载。

**Q：VirtualAPK 对于 plugin 中 BroadcastReceiver 的代理流程。**

A：在解析 plugin apk 时，将所有 receiver 动态注册。具体是在 LoadedPlugin 的构造方法中执行的。

