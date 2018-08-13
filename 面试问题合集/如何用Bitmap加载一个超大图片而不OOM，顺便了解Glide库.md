# 如何用Bitmap加载一个超大图片而不 OOM，顺便了解 Glide 库

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