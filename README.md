# Bitmap-Loading-And-Cache
# Bitmap的加载和Cache

## Bitmap的高效加载
Bitmap 在Android中指的是一张照片，可以是Png格式也可以是Jpg等其他常见的图片格式。那么如何加载一个图片呢？
BitmapFactory类提供了四种方法：
* decodeFile 从文件系统
* decodeResource 从资源
* decodeStream 从输入流
* decodeByteArray 从字节数组

其中decodeFile 和 decodeResource又间接调用了decodeStream方法,这四种方法最终是在Android的底层实现的。对应着BitmapFactory类中的几个native方法。

**高效加载Bitmap的实现 其核心思想很简单：**

通过BitmapFactory.Options 使用其 inSampleSize(采样率)来缩放大图片。通过inSampleSize缩放后可以降低内存的占用从而在一定程度上避免OOM(Out Of Memory Error),提高Bitmap加载时的性能。


>附上一段代码(Button按钮点击,ImageView加载资源图片 手机：小米NOTE Pro)

```
@Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_bitmap);
        mImageView = (ImageView) findViewById(R.id.imageView);
        Point point = new Point();
        getWindowManager().getDefaultDisplay().getSize(point);
        mScreenWidth = point.x;     // 屏幕宽度 1440
        mScreenHeight = point.y;    // 屏幕高度 2560

        Log.e("BitmapActivity", String.valueOf(mScreenHeight) + "," + String.valueOf(mScreenWidth));
    }

 public void loading(View view) {
        BitmapFactory.Options options= new BitmapFactory.Options();
        
        // options.inJustDecodeBounds设置为true,不会真正的将图片加载到内存中去
        options.inJustDecodeBounds = true;

        BitmapFactory.decodeResource(getResources(), R.mipmap.dog, options);


        int outHeight = options.outHeight;  // 图片的高度 3200
        int outWidth = options.outWidth;    // 图片的宽度 2400

        int heightScale = outHeight / mScreenHeight; // 高度比例 
        int widthScale = outWidth / mScreenWidth;   // 宽度比例
        int scale = widthScale > heightScale ? widthScale : heightScale;

        // 将算好的采样率设置
        options.inSampleSize = scale;
        options.inJustDecodeBounds = false;

        Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.mipmap.dog, options);
        mImageView.setImageBitmap(bitmap);
    }
```

![Bitmap-dog](http://a1.qpic.cn/psb?/V14YlNrL2eQEkW/NPuxvG1sl*f5KAB*tasYERTgI8D9BzikSq*RqvDI83g!/b/dLEAAAAAAAAA&bo=hQLzAYUC8wEDByI!&rf=viewer_4)

![Bitmap-ui](http://a3.qpic.cn/psb?/V14YlNrL2eQEkW/CwrQ0EoSp7nGtM.CtxPU9yGSsrzd*jUlnYWPXSYDuX8!/b/dAEBAAAAAAAA&bo=BwKjAwcCowMDACU!&rf=viewer_4)

## Android的缓存策略

> 缓存策略在Android中有着广泛的使用场景。但为了避免下载图片消耗过多的流量，缓存策略此时就变的很重要。当程序第一次从网络加载图片后，将其缓存到存储设备上，下次使用的时候就不必再从网络拉取，为了提高用户体验，往往还会降图片再在内存中缓存一份，这样当应用打算从网络请求一张图片的时候，会先从内存中获取，内存没有就从存储设备获取，存储设备没有就在从网络上拉取。


LRU算法(*Least Recently Used*):近期最少使用算法。其核心思想是当缓存满时，会优先淘汰那些近期最少使用的缓存对象。
LRU算法缓存有两种：*LruCache*(内存缓存) 和 *DishLruCache*(存储设备缓存),通过二者的完美结合，就能实现一个具有很高使用价值的ImageLoader。

### 内存缓存LruCache

*LruCache* 是一个线程安全的广泛类,其内部采用一个*LinkedHashMap*以强引用的方式存储外界的缓存对象，提供了get和put方法来完成缓存的获取和添加操作，当缓存满时，LruCache会移除较早使用的缓存对象，然后再添加新的缓存对象。

* 强引用：直接的对象引用;
* 软引用：当一个对象只有软引用存在时，系统内存不足时此对象会被gc回收;
* 弱引用：当一个对象只有弱引用存在时，此对象会随时被gc回收;

*LruCache*的定义:

```
public class LruCache<K,V>{
    private final LinkedHashMap<K,V> map;
    ...
}
```

* *LruCache*的实现也很简单，附上LruCache的典型初始化过程:

```
int maxMemory = (int) Runtime.getRuntime().maxMemory();
int cacheSise =maxMemory / 8;
mLruCache = new LruCache<String, Bitmap>(cacheSize) {
    //计算缓存对象的大小
    @Override
    protected int sizeOf(String key, Bitmap bitmap) {
        // 单位是MB
        return bitmap.getRowBytes() * bitmap.getHeight() / bitmap.getByteCount();
    }

    //移除旧缓存时调用
    @Override
    protected void entryRemoved(boolean evicted, String key, Bitmap oldValue, Bitmap newValue) {
        super.entryRemoved(evicted, key, oldValue, newValue);
        //  资源回收的工作
    }
};

```

* LruCache的添加和获得一个缓存对象也很简单:

```
mMemoryCache.get(key);  // 获得缓存
mMemoryCache.put(key , bitmap);     // 添加
mMemoryCache.remove(key);   // 删除
```

### 存储设备缓存DiskLruCache

*DiskLruCache* 磁盘缓存：通过将缓存对象写入文件系统从而实现缓存的效果。
DiskLruCache得到了Android官方文档的推荐，但不属于AndroidSDK的一部分，源码可从如下网址得到

```
https://android.googlesource.com/platform/libcore/+/android-4.1.1_r1/luni/src/main/java/libcore/io/DiskLruCache.java
```

* DiskLruCache的创建


```
private static final long DISK_CACHE_SIZE = 1024 * 1024 * 50;//磁盘缓存50M大小

File diskCacheDir = getExternalCacheDir();
if (!diskCacheDir.exists()){
    diskCacheDir.mkdirs();
}
/**
 * Opens the cache in {@code directory}, creating a cache if none exists
 * there.
 *
 * @param directory a writable directory 缓存目录
 * @param appVersion 应用版本号
 * @param valueCount the number of values per cache entry. Must be positive.
 * @param maxSize the maximum number of bytes this cache should use to store
 * @throws IOException if reading or writing the cache directory fails
 */
DiskLruCache diskLruCache = DiskLruCache.open(diskCacheDir, 1, 1, DISK_CACHE_SIZE);

```
* *DiskLruCache*的缓存添加

*DiskLruCache*缓存添加是通过*Editor*完成的。当使用*Editor*编辑完缓存对象后，记得使用*commit()*.

**注意**：图片的url可能含有特殊字符，所以一般采用url的md5值做为key。


```
String imgUrl = "";
String key = MD5Util.getMd5Value(imgUrl);
DiskLruCache.Editor editor = diskLruCache.edit(key);
if (editor != null) {
    editor.getString(0);
}
editor.commit();//提交
editor.abort();//回退
diskLruCache.flush();//刷新

```

* *DiskLruCache*的缓存查找

通过*DiskLruCache.get*获取一个*SnapShot*对象，再使用*Snapshot*对象即可获得缓存的文件输入流。

**关于FileInputStream(文件描述符)**：直接使用*BitmapFactory.Options* 对*FileInputStream*进行缩放会出现问题，因为*FileInputStream*是一种有序的文件流，两次decodeStream调用会影响文件流的位置属性，导致第二次decodeStream时得到的是null。为了解决这个问题，可以通过文件流来得到它所对应的文件描述符，然后通过BitmapFactory.decodeFileDescriptor来加载一张缩放后的图片。


附上一段代码：
```
String key = MD5Util.getMd5Value(imgUrl);

Bitmap bitmap = null;

DiskLruCache.Snapshot snapshot = diskLruCache.get(key);
if (snapshot != null) {
    FileInputStream fileInputStream = (FileInputStream) snapshot.getInputStream(DISK_CACHE_INDEX);
    //从文件流中获取文件描述符
    FileDescriptor fileDescriptor = fileInputStream.getFD();
    bitmap = decodeSampledBitmapFromFileDescriptor(fileDescriptor,reqWidth,reqHeight);
    if (bitmap != null) {
        addBitmapToMemoryCache(key,bitmap);
    }
}
```

* 最后附上自写的ImageLoader

### [JYImageLoader](https://github.com/JiaYang627/JYImageLoader)
