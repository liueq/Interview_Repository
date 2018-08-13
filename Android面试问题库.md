## 1. synchronized 关键字用法

> 未查找资料时，自己能够立即想到的内容如下

```
两种用途，一是修饰方法，二是修饰代码块。

当修饰方法的时候，进入该方法时会获取锁，退出时释放锁。所获取锁的类型，如果是静态方法，那么锁是class，如果是非静态方法，锁是对象锁。未获取锁想要访问该方法时，会阻塞直到能够获取锁为止。修饰方法的时候，锁的粒度相对较大，容易造成资源浪费。

当修饰代码块的时候，需要显示指定一个对象锁，进入该代码块需要获得该锁。注意可以使用 this，.class 等。相较于修饰方法，更节省资源。

在某个版本之前，synchronized 比起 ReentraientLock 更耗资源，但是在某个版本之后（记不清哪个了）。synchronized 经过改善，现在除非确实需要用 ReentraintLock 来划分更细的粒度，否则不在推荐使用 ReentraintLock 替代 synchronized。
```

> 经过修正后

```
两种用途，一是修饰方法，二是修饰代码块。

当修饰方法的时候，进入该方法时会自动获取锁，退出或跑出异常时自动释放锁。所获取锁的类型，如果是静态方法，那么锁是class，如果是非静态方法，锁是对象锁。未获取锁想要访问该方法时，会阻塞直到能够获取锁为止。修饰方法的时候，锁的粒度相对较大，容易造成资源浪费。

当修饰代码块的时候，需要显示指定一个对象锁，进入该代码块需要获得该锁。注意可以使用 this，.class 等。相较于修饰方法，更节省资源。

JVM 可以优化 synchronized，将两个相邻的使用相同锁的 synchronzied 可以合并为一个，实现锁的粗化，减少释放，获取锁的次数，提升性能。

synchronized 是虚拟机层面实现的，而 ReentrantLock 是JDK 实现。

在JDK 1.6之前，synchronized 比起 ReentrantLock 更耗资源，但是在JDK1.6之后通过使用偏向锁，CAS等两者性能差不多。synchronized 经过改善，现在除非确实需要用 ReentrantLock 来划分更细的粒度，否则不在推荐使用 ReentrantLock 替代 synchronized。

synchronized 范围的的变量对其他线程来说是可见的。
由于JVM 的优化，即使不使用 synchronized，只要cpu有空闲时间，就会同步主存和线程的工作内存的变量值。
（线程对于变量的读写都是在自己的工作内存中，而线程之间的通信则是通过同步自己的工作内存和主存实现的。
```

## 2. volatile 用法

> 自己立即想到的内容

```
volatile 所修饰的 field，对于所有线程都是可见的，总能返回最新值。

volatile 可以阻止JVM 重排序。
```

> 修正后

```
volatile 所修饰的 field，对于所有线程都是可见的，总能返回最新值。但是不具有原子性。

volatile 可以阻止JVM 重排序。

当一个变量没有涉及到其他状态变量的不变约束时，才声明为 volatile，否则是 synchronized。

原理是使用 memory barrier 来刷新缓存。

volatile 用来修饰数组的时候，仅仅是对数组的引用提供了 volatile 语义，其成员并没有。
```

## 3. 动态权限适配，权限组概念

自 Android M 开始，某些重要权限除了在 AndroidManifest 中定义外，还需要在代码中动态申请权限。一共有24个这样的权限，分为了6个权限组。

在Android O 以前，只要申请了权限组中的任意一个权限，那么就自动获取了该权限组中的所有权限，无需再申请。

在Android O 以后，即使申请了权限组中的某一个权限，仍然需要申请其他权限才可使用，只是申请其他权限的时候，会被自动授权，无需用户再次操作。

其他一些危险权限或者辅助功能，需要在设置中手动打开，可以通过Intent 引导用户。例如 WRITE_SETTINGS, SYSTEM_ALERT_WINDOW 等。

| 权限组        | 权限                     |
| ---------- | ---------------------- |
| CALENDAR   | READ_CALENDAR          |
|            | WRITE_CALENDAR         |
| CAMERA     | CAMERA                 |
| CONTACTS   | READ_CONTACTS          |
|            | WRITE_CONTACTS         |
|            | GET_ACCOUNTS           |
| LOCATION   | ACCESS_FINE_LOCATION   |
|            | ACCESS_COARSE_LOCATION |
| MICROPHONE | RECORD_AUDIO           |
| PHONE      | READ_PHONE_STATE       |
|            | CALL_PHONE             |
|            | READ_CALL_LOG          |
|            | WRITE_CALL_LOG         |
|            | ADD_VOICEMAIL          |
|            | USE_SIP                |
|            | PROCESS_OUTGOING_CALLS |
| SENSORS    | BODY_SENSORS           |
| SMS        | SEND_SMS               |
|            | RECEIVE_SMS            |
|            | READ_SMS               |
|            | RECEIVE_WAP_PUSH       |
|            | RECEIVE_MMS            |
| STORAGE    | READ_EXTERNAL_STORAGE  |
|            | WRITE_EXTERNAL_STROAGE |

## 4. 网络请求缓存处理， OKHttp 如何处理的缓存

### 4.1 WebVew 的缓存机制

WebView 中一共有5种缓存机制：

1. 浏览器缓存，WebView 默认实现，无法修改。
2. Application Cache 缓存。可以支持 Application Cache API 的换粗，主要适用于 Html5应用。
3. Dom Storage 缓存。
4. Web SQL Database 缓存。
5. Indexed Database 缓存。

#### 4.1.1 Application Cache 使用方法

```java
// 通过设置WebView的settings来实现
WebSettings settings = getSettings();

// 1. 设置缓存路径
String cacheDirPath = context.getFilesDir().getAbsolutePath()+"cache/";
settings.setAppCachePath(cacheDirPath);

// 2. 设置缓存大小
settings.setAppCacheMaxSize(20*1024*1024);

//3. 开启Application Cache存储机制
settings.setAppCacheEnabled(true);

// 特别注意
// 每个 Application 只调用一次 WebSettings.setAppCachePath() 和 WebSettings.setAppCacheMaxSize()
```

#### 4.1.2 setCacheMode 4种设置的实际效果

* LOAD_DEFAULT 使用默认设置，根据服务器端的Cache-Control ，主要是 Cookie 的 Expires设置。
* LOAD_CACHE_ELSE_NETWORK 如果存在缓存，那么只读取缓存。
* LOAD_NO_CACHE 总是从网络获取，该设置才能够让返回上一页的时候立即刷新页面。
* LOAD_CACHE_ONLY 只读取缓存。

关于缓存，主要还是结合服务器端进行设置，而且和浏览器的表现似乎还不一样。

对于静态图片这样的资源，即使 LOAD_NO_CACHE，仍然会读取缓存。只有第一次访问会从网站获取，之后就是缓存了。所以对于图片这样的资源来说，文件名必须要有时间戳，否则同名图片发生了改变，浏览器不会及时获取最新的图片。

使用 reload() 方法，总能够重新刷新资源。

#### 4.1.3 如何清除 setCacheMode 缓存的数据。

删除缓存目录的文件即可，缓存目录可以通过 `getCacheDir()` 得到。

#### 4.1.4 WebView + OkHttp 实现单独的资源缓存机制

由于 WebView 自己的缓存机制有大小限制，可以在请求某些图片等较大资源的时候，单独将文件缓存，下次就不用再请求了。

关键方法: `shouldInterceptRequest`

此方法的 super 默认返回 null，这样WebView 就会从网络获取资源。因此拦截的关键就是在这里构造一个 WebResourceResponse 对象并返回。关键是 WebResourceResponse 的内容如何获取？由于WebView 无法拿到单独资源，所以应该要使用 OkHttp单独进行资源的请求并缓存。

```java
@Override
        public WebResourceResponse shouldInterceptRequest(WebView view, WebResourceRequest request) {
            Log.d(TAG, "shouldInterceptRequest: " + request.getUrl());
            String url = request.getUrl().toString();
            if (url.endsWith("png")) {
                try {
                    WebResourceResponse cached = getCache(url);
                    if (cached == null) {
                        // okhttp to reach data
                        Log.d(TAG, "shouldInterceptRequest: 为获取缓存，使用 OkHttp");
                        okHttp(url);
                        cached =  getCache(url);
                    } else {
                        Log.d(TAG, "shouldInterceptRequest: 读取缓存");
                    }

                    return cached;
                } catch (FileNotFoundException e) {
                    e.printStackTrace();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
            return super.shouldInterceptRequest(view, request);
        }
```

**疑问**

目前还是不懂当 Html 中嵌入有资源时，请求是如何分发的，从抓包来看，只有两个http 包，分别是:html，网站icon。并没有html 中所包含的图片的请求。但是从Android端看，确实是有这样一个 request。暂时还不懂Http更多的东西，先不管吧。

### 4.2 OkHttp的缓存机制

#### 4.2.1 OkHttp 缓存机制在 API 层面的应用

> OkHttpClient 需要设置 Cache

```java
private Cache createCache() {
    File cachePath = ApiToolsApplication.getInstance().getCacheDir();
    int cacheSize = 10 * 1024 * 1024;//10MB
    Cache cache = new Cache(new File(cachePath, "okhttp-cache"), cacheSize);
    try {
        cache.evictAll();
    } catch (IOException e) {
        e.printStackTrace();
    }
    return cache;
}

OkHttpClient.Builder builder = new OkHttpClient.Builder();
builder.cache(createCache()); //需要这里设置 cache 才能缓存。光设置CacheControl没用。
mClient = builder.build();
```

> 客户端可设置的几种 Cache-Control

```java
CacheControl.Builder cacheBuilder = new CacheControl.Builder();
cacheBuilder.noCache(); //总是不使用缓存
cacheBuilder.onlyIfCached(); //如果存在缓存，就是用缓存，否则返回504

//无论客户端设置如何，都必须要服务器设置Cache-Control才行。
//下面几组可以定义具体时间的，虽然基本上有效，但是时间却有些对不上，这里需要看源代码才能搞清楚。
cacheBuilder.maxAge(30, TimeUnit.SECONDS); //接受缓存的最长存活时间，如果超过了该时间，及时为过期，仍然访问网络。
cacheBuilder.minFresh(30, TimeUnit.SECONDS);//缓存的最小有效时间，超过该时间，缓存过期
cacheBuilder.maxStale(30, TimeUnit.SECONDS); //缓存过期后，仍可用的最大时间
CacheControl cacheControl = cacheBuilder.build();
return cacheControl;
```

> 在 Request 中添加 Cache-Control header

```java
Request.Builder builder = new Request.Builder();
builder.cacheControl(createCacheControl());
```

#### 4.2.2 了解 OkHttp 缓存机制的实现原理

查看笔记: 

* CacheInterceptor详解.md
* CacheStrategy详解.md
* Cache详解.md

#### 4.2.3 自己辅助实现 OkHttp 的缓存

感觉没有必要。自己实现虽然可以使用 DiskLruCache 来提升LRU 算法的性能，但是根据URL缓存的逻辑还不一定比OkHttp 自身更好。

## 5. 如何用Bitmap加载一个超大图片而不 OOM，顺便了解 Glide 库

### 5.1 自己写代码实现加载大图

1. 对于本地大图的加载，包括缩略图生成等。
2. 对于网络大图的加载。

不论本地图片或者网络图片，只是来源不同，主要的方法都一样。其中网络图片，如果是用 OkHttp 进行的网络连接，那么注意 body 只能够读取一次。因此无法使用 inJustBounds 参数，如果使用了，没法第二次真正的读取文件。

> 1 加载图片时，首先要知道应用 OOM 的界限。

```java
//获取 App 的最大可用内存
int maxMemory = (int) (Runtime.getRuntime().maxMemory() / 1024);
```

> 2 根据所要加载的图片和装载 ImageView的大小，计算出 sample size 进行压缩。

```java
BitmapFactory.Options options = new BitmapFactory.Options();
            options.inJustDecodeBounds = true;
            BitmapFactory.decodeFile(img.getAbsolutePath(), options);
			//inJustDecodeBounds 能够获取的数据只有3个
            int width = options.outWidth;
            int height = options.outHeight;
            String outMimeType = options.outMimeType;
            {
                Log.d(TAG, "doInBackground: 图片默认宽度： " + width);
                Log.d(TAG, "doInBackground: 图片默认高度： " + height);
                Log.d(TAG, "doInBackground: 图片MIME: " + outMimeType);
            }

            int targetWidth = mIv.getWidth();
            int targetHeight = mIv.getHeight();
            {
                Log.d(TAG, "doInBackground: ImageView 固定宽度： " + targetWidth);
                Log.d(TAG, "doInBackground: ImageView 固定高度: " + targetHeight);
            }

            int inSampleSize = 1;

			//计算 sample size，使用宽高中较小者
            if (width > targetWidth || height > targetHeight) {
                int widthRatio = Math.round((float) width / (float) targetWidth);
                int heightRatio = Math.round((float) height / (float) targetHeight);

                inSampleSize = widthRatio < heightRatio ? widthRatio : heightRatio;
            }

            options.inSampleSize = inSampleSize;
            options.inJustDecodeBounds = false;
            Bitmap bitmap = BitmapFactory.decodeFile(img.getAbsolutePath(), options);
```

#### 5.1.1 使用 RecyclerView + LruCache 加载一个图片列表

> 1 LruCache 的使用

必须要覆盖 *sizeOf* 方法， 这样才能够正确的计算 LruCache 的大小，否则 cache.size() 只能返回对象数量。

设置LruCache 的最大值最好都统一用 byte，否则换算来换算去很麻烦。

```java
LruCache<String, Bitmap> mCache = new LruCache<String, Bitmap>(maxMemo / 10) {
	@Override
	protected int sizeOf(String key, Bitmap value) {
		return value.getByteCount();
	}
};
```

> 2 RecyclerView 延迟加载的要点

onBindViewHolder 被调用的时候，会将数据显示在 Item 上，此时如果数据没准备好，应该先是一个 PlaceHolder。

```java
@Override
public void onBindViewHolder(ViewHolder holder, int position) {
    String url = mUrls.get(position);
    Bitmap bitmap = mCache.get(url);
    if (bitmap == null) {
        bitmap = mHolderBitmap;
        // 这里告知需要下载图片，暂时用 Holder 图片代替
        loadBitmap(url);
    }
    holder.mImageView.setImageBitmap(bitmap);
    holder.mImageView.setTag(url); //将 ViewHolder 中的 View 和 url 进行一个绑定，以便数据可显示时，ViewHolder 绑定的还是否是该View。
    mUrlToViewHolder.put(url, holder); //记录到Map 中
}
```

当 *onViewRecycled* 被调用时，该 holder 就被回收了，此时应该清理不使用的对象。

```java
@Override
public void onViewRecycled(ViewHolder holder) {
    super.onViewRecycled(holder);
    //移除 ImageView 中的 bitmap，减少内存使用
    holder.mImageView.setImageBitmap(null); //从View上移除 bitmap，实际上该 bitmap 还在 mCache。只让 mCache 持有引用，以便在内存不够时，可以直接删除。
    //移除 Listener
    String url = (String) holder.mImageView.getTag(); //获取该 holder 对应的数据，取消对应的后台操作
    ImageDownloader.getInstance().cancel(url);
    mUrlToDownloadListeners.remove(url);
}
```

当后台操作完成，可以显示在 Item 上时，应该要判断该 View 是否还在可见区域内，即该 ViewHolder 是否还是绑定的该 View。

```java
private void loadBitmap(String url) {
    ImageDownloader.ImageDownloadListener listener = new ImageDownloader.ImageDownloadListener() {
        @Override
        public void onLoaded(String url, File file) {
            BitmapFactory.Options options = new BitmapFactory.Options();
            options.inSampleSize = 8;
            final Bitmap bitmap = BitmapFactory.decodeFile(file.getAbsolutePath(), options);
            mCache.put(url, bitmap);
            // 如果 View 还在可视区域内，那么加载图片(当Holder被复用时，url 就发生了变化，可以用来判断)
            final ViewHolder holder = mUrlToViewHolder.get(url); 
            if (url.equals(holder.mImageView.getTag())) {
              //必须要回到主线程，否则setImageBitmap 不会更新界面
                TheApplication.instance().backMainThread(new Runnable() {
                    @Override
                    public void run() {
                        holder.mImageView.setImageBitmap(bitmap);
                    }
                });
            }
        }
    };
    mUrlToDownloadListeners.put(url, listener);
    ImageDownloader.getInstance().addDownloadTask(url, listener);
}
```

### 5.2 Glide 库的使用

//TODO 暂时没兴趣

### 5.3 Glide 库的源码大致了解

//TODO 暂时没兴趣

### 5.4 超大图片的局部显示

使用 BitmapRegionDecoder 来加载局部的图片。

```java
File file = new File(Environment.getExternalStorageDirectory(), "biggest_pi
BitmapRegionDecoder decoder = null;
try {
  //使用 newInstance 的方式创建 Decoder
    decoder = BitmapRegionDecoder.newInstance(file.getAbsolutePath(), false
} catch (IOException e) {
    e.printStackTrace();
}
int zoomX = (int) x * mSampleX;
int zoomY = (int) y * mSampleY;
int left = zoomX - mViewWidth / 2;
int right = zoomX + mViewWidth / 2;
int top = zoomY - mViewHeight / 2;
int bottom = zoomY + mViewHeight / 2;
mZoomRect.set(left, top, right, bottom);
BitmapFactory.Options options = new BitmapFactory.Options();
//关键在于 decodeRegion，只加载 Rect 部分的图片，是相对于图片本身的长宽而言。
Bitmap bitmap = decoder.decodeRegion(mZoomRect, options);
```

## 6 进程保活

### 6.1 进程优先级概念

优先级由高到低：

* 前台进程（Foreground Process）
* 可见进程（Visible Process）
* 服务进程（Service Process）
* 后台进程（Background Process）
* 空进程（Empty Process）

#### 6.1.1 前台进程

用户当前操作所必须的进程，一般来说数量不多。之后当内存不足以支持它们同时继续运行才会终止。

以下几种情况就是前台进程：

* 进程拥有用户正在交互的 Activity （onResume被调用）。
* 进程拥有某个 Service，该 Service 被用户正在交互的 Activity 所绑定。
* 进程拥有调用了 *startForeground()* 的 Service。
* 进程拥有正在执行生命周期回调的 Service（onCreate, onStart, onDestroy）。
* 进程拥有正在执行 *onReceive()* 方法的 BroadcastReceiver。

#### 6.1.2 可见进程

没有任何前台组件，但是会影响用户屏幕上可见内容的进程。包括以下两种情况：

* 拥有不在前台，但是对用户可见的 Activity（onPause 被调用）。
* 拥有绑定到前一个条件中的 Activity 的Service。

#### 6.1.3 服务进程

没有关系到用户可见内容，但是仍然在执行一些重要的操作。

根据是否调用 *startService()* 判断，且不在上述两个进程中。

#### 6.1.4 后台进程

退到后台的 Activity（调用了 onStop），对用户不可见，根据系统 LRU 算法，内存不够时随时可以被杀死。

#### 6.1.5 空进程

不包含任何活动组件的进程，存在的意义在作为缓存。

### 6.2 内存回收策略

根据进程的 oom_adj 值，来决定在内存回收时候的优先级。

oom_adj 在 *ProcessList.java* 定义。

oom_adj 的优先级基本上和进程优先级差不多，只是有更细粒度的划分，以及系统应用的更高优先级等。

具体的值不在这里赘述，需要时查阅即可。

### 6.3 保活措施实践

#### 6.3.1 1像素 Activity

监听屏幕开闭，在锁屏时在前台展示一个 1px Activity，亮屏时，关闭。

> 监听广播

```java
IntentFilter filter = new IntentFilter();
filter.addAction("android.intent.action.SCREEN_OFF");
filter.addAction("android.intent.action.SCREEN_ON");
registerReceiver(mReceiver, filter);
```

> AndroidManifest 中 Activity 的设定

注意 theme 要选择透明，launchMode 必须是 singleInstance，并且从RecentTasks 中排除。

```xml
<activity
            android:name=".OnePixelActivity"
            android:theme="@style/AppThemeTranslucent"
            android:launchMode="singleInstance"
            android:excludeFromRecents="true"
            android:finishOnTaskLaunch="false"/>
```

> Activity 设置

在 Activity 中设置 Window 1px。

```java
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
  
    Window window = getWindow();
    window.setGravity(Gravity.LEFT|Gravity.TOP);
    WindowManager.LayoutParams params = window.getAttributes();
    params.x = 0;
    params.y = 0;
    params.width = 1;
    params.height = 1;
    window.setAttributes(params);
}
```

#### 6.3.2 InnerService

在 Service 中，调用 startForeground 可以将 oom_adj 优先级提升到 2，但是会在通知栏创建一个 Notification。

现在创建一个 InnerService，和OuterService 使用相同的 Notificaiton ID，都调用 startForeground 方法。然后再停止 InnerService。这样 Notification 会消失，并且 OuterService 仍然是 foreground 状态。

> OuterService 设定

```java
@Override
public void onCreate() {
    super.onCreate();
    Log.d(TAG, "onCreate: ");
    startForeground(NID, getNotification());
}
```

> InnerService 设定

```java
@Override
public void onCreate() {
    super.onCreate();
    Log.d(TAG, "onCreate: ");
    startForeground(NID, getNotification()); //保证 id 和 outer 相同
}
```

#### 6.3.4 监听广播拉活

常用的广播：

* RECEIVE_BOOT_COMPLETE **开机广播**
* ACCESS_NETWORK_STATE **网络变化**
* CHANGE_NETWORK_STATE
* ACCESS_WIFI_STATE
* CHANGE_WIFI_STATE
* ACCESS_FINE_LOCATION
* ACCESS_LOCATION_EXTRA_COMMANDS
* MOUNT_UNMOUNT_FILESYSTEMS **文件挂载**
* SCREEN_ON **屏幕亮灭**
* SCREEN_OFF
* RECEIVE_USER_PRESENT **屏幕解锁**
* PACKAGE_ADDED **应用安装卸载**
* PACKAGE_REMOVED

#### 6.3.5 STICK 保活

在 onStartCommand 方法的返回值中，使用 START_STICK。在进程非 force stop 的情况下异常退出后，会尝试再次启动。注意此时的 intent 仍然是上一次的一致。在目前尝试的手机中，只会启动一次。

启动频率按照官方的说法是：5， 10， 20， 杀死5次以上不再拉起。但是实际上两次就不会拉起了。

#### 6.3.6 Native 进程

Native 进程不受 Android 进程管理的限制。因此可以用来拉起应用。但是在 5.0之后，杀死 app 时，会同时杀死所有的 native 进程，此方法不再有效。

#### 6.3.6 JobScheduler

使用 JobScheduler 可以设置定时任务，但是注意，JobScheduler 的时间并不是精确的，而是会根据系统空闲窗口来调整。

在 JobScheduler 中执行的代码是在主线程。

>  AndroidManifest 定义

注意权限的声明要在 service 中。

```xml
<service android:name=".JobSchedulerService"
                 android:permission="android.permission.BIND_JOB_SERVICE" />
```

> 创建任务

```java
JobScheduler jobScheduler = (JobScheduler) getSystemService(JOB_SCHEDULER_SERVICE);
ComponentName componentName = new ComponentName("com.liueq.testprocessalive", "com.liueq.testprocessalive.JobSchedulerSe
JobInfo.Builder builder = new JobInfo.Builder(1234, componentName);
builder.setPeriodic(1000);
jobScheduler.schedule(builder.build());
```

#### 6.3.7 其他措施

* 建立一个账号，利用账号同步机制
* 通知管理权限，或者辅助功能开启。
* 接入一些 SDK。

## 7 ListView 图片加载错乱原因及解决办法

异步加载的时候，当图片下载完成，此时对应的 View 已经被回收了，所以造成了错乱。

其实可以根据这个问题来深入 ListView 的机制，但是实在是不想看，太麻烦了。

//TODO
