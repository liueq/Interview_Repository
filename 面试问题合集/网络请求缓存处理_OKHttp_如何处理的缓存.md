# 网络请求缓存处理， OKHttp 如何处理的缓存

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

- LOAD_DEFAULT 使用默认设置，根据服务器端的Cache-Control ，主要是 Cookie 的 Expires设置。
- LOAD_CACHE_ELSE_NETWORK 如果存在缓存，那么只读取缓存。
- LOAD_NO_CACHE 总是从网络获取，该设置才能够让返回上一页的时候立即刷新页面。
- LOAD_CACHE_ONLY 只读取缓存。

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

- CacheInterceptor详解.md
- CacheStrategy详解.md
- Cache详解.md

#### 4.2.3 自己辅助实现 OkHttp 的缓存

感觉没有必要。自己实现虽然可以使用 DiskLruCache 来提升LRU 算法的性能，但是根据URL缓存的逻辑还不一定比OkHttp 自身更好。