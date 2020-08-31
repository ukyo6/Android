## 一、谈谈Glide

### 1.1 Glide 使用有多简单？

Glide由于其口碑好，很多开发者直接在项目中使用，使用方法相当简单

[https://github.com/bumptech/glide](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fbumptech%2Fglide)

1、添加依赖：

```bash
implementation 'com.github.bumptech.glide:glide:4.10.0'
annotationProcessor 'com.github.bumptech.glide:compiler:4.10.0'
```

2、添加网络权限

```xml
<uses-permission android:name="android.permission.INTERNET" />
```

3、一句代码加载图片到ImageView

```css
Glide.with(this).load(imgUrl).into(mIv1);
```

进阶一点的用法，参数设置

```csharp
RequestOptions options = new RequestOptions()
            .placeholder(R.drawable.ic_launcher_background)
            .error(R.mipmap.ic_launcher)
            .diskCacheStrategy(DiskCacheStrategy.NONE)
            .override(200, 100);
    
Glide.with(this)
            .load(imgUrl)
            .apply(options)
            .into(mIv2);
```

使用Glide加载图片如此简单，这让很多开发者省下自己处理图片的时间，图片加载工作全部交给Glide来就完事，同时，很容易就把图片处理的相关知识点忘掉。

### 1.2 为什么用Glide？

从前段时间面试的情况，我发现了这个现象：简历上写熟悉Glide的，基本都是熟悉使用方法，很多3年-6年工作经验，除了说Glide使用方便，不清楚Glide跟其他图片框架如**Fresco**的对比有哪些优缺点。

首先，当下流行的图片加载框架有那么几个，可以拿 Glide 跟[Fresco](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Ffacebook%2Ffresco)对比，例如这些：

**Glide：**

- 多种图片格式的缓存，适用于更多的内容表现形式（如Gif、WebP、缩略图、Video）
- 生命周期集成（根据Activity或者Fragment的生命周期管理图片加载请求）
- 高效处理Bitmap（bitmap的复用和主动回收，减少系统回收压力）
- 高效的缓存策略，灵活（Picasso只会缓存原始尺寸的图片，Glide缓存的是多种规格），加载速度快且内存开销小（默认Bitmap格式的不同，使得内存开销是Picasso的一半）

**Fresco：**

- 最大的优势在于5.0以下(最低2.3)的bitmap加载。在5.0以下系统，Fresco将图片放到一个特别的内存区域(Ashmem区)
- 大大减少OOM（在更底层的Native层对OOM进行处理，图片将不再占用App的内存）
- 适用于需要高性能加载大量图片的场景

对于一般App来说，Glide完全够用，而对于图片需求比较大的App，为了防止加载大量图片导致OOM，Fresco 会更合适一些。并不是说用Glide会导致OOM，Glide默认用的内存缓存是LruCache，内存不会一直往上涨。

## 二、假如让你自己写个图片加载框架，你会考虑哪些问题？

首先，梳理一下必要的图片加载框架的需求：

- 异步加载：线程池
- 切换线程：Handler，没有争议吧
- 缓存：LruCache、DiskLruCache
- 防止OOM：软引用、LruCache、图片压缩、Bitmap像素存储位置
- 内存泄露：注意ImageView的正确引用，生命周期管理
- 列表滑动加载的问题：加载错乱、队满任务过多问题

当然，还有一些不是必要的需求，例如加载动画等。

### 2.1 异步加载：

线程池，多少个？

缓存一般有三级，内存缓存、硬盘、网络。

由于网络会阻塞，所以读内存和硬盘可以放在一个线程池，网络需要另外一个线程池，网络也可以采用Okhttp内置的线程池。

读硬盘和读网络需要放在不同的线程池中处理，所以用两个线程池比较合适。

Glide 必然也需要多个线程池，看下源码是不是这样

```java
public final class GlideBuilder {
  ...
  private GlideExecutor sourceExecutor; //加载源文件的线程池，包括网络加载
  private GlideExecutor diskCacheExecutor; //加载硬盘缓存的线程池
  ...
  private GlideExecutor animationExecutor; //动画线程池
```

Glide使用了三个线程池，不考虑动画的话就是两个。

### 2.2 切换线程：

图片异步加载成功，需要在主线程去更新ImageView，

**无论是RxJava、EventBus，还是Glide，只要是想从子线程切换到Android主线程，都离不开Handler。**

看下Glide 相关源码：

```java
    class EngineJob<R> implements DecodeJob.Callback<R>,Poolable {
      private static final EngineResourceFactory DEFAULT_FACTORY = new EngineResourceFactory();
      //创建Handler
      private static final Handler MAIN_THREAD_HANDLER =
          new Handler(Looper.getMainLooper(), new MainThreadCallback());
```

> 问RxJava是完全用Java语言写的，那怎么实现从子线程切换到Android主线程的？  依然有很多3-6年的开发答不上来这个很基础的问题，而且只要是这个问题回答不出来的，接下来有关于原理的问题，基本都答不上来。
>
> 有不少工作了很多年的Android开发不知道**鸿洋、郭霖、玉刚说**，不知道掘金是个啥玩意，内心估计会想是不是还有叫掘银掘铁的（我不知道有没有）。
>
> 我想表达的是，干这一行，真的是需要有对技术的热情，不断学习，**不怕别人比你优秀，就怕比你优秀的人比你还努力，而你却不知道**。

### 2.3 缓存

我们常说的图片三级缓存：内存缓存、硬盘缓存、网络。

#### 2.3.1 内存缓存

一般都是用`LruCache`

Glide 默认内存缓存用的也是LruCache，只不过并没有用Android SDK中的LruCache，不过内部同样是基于LinkHashMap，所以原理是一样的。

```csharp
// -> GlideBuilder#build
if (memoryCache == null) {
  memoryCache = new LruResourceCache(memorySizeCalculator.getMemoryCacheSize());
}
```

既然说到LruCache ，必须要了解一下LruCache的特点和源码：

**为什么用LruCache？**

LruCache 采用**最近最少使用算法**，设定一个缓存大小，当缓存达到这个大小之后，会将最老的数据移除，避免图片占用内存过大导致OOM。

##### LruCache 源码分析

```cpp
    public class LruCache<K, V> {
    // 数据最终存在 LinkedHashMap 中
    private final LinkedHashMap<K, V> map;
    ...
    public LruCache(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }
        this.maxSize = maxSize;
        // 创建一个LinkedHashMap，accessOrder 传true
        this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
    }
    ...
```

LruCache 构造方法里创建一个**LinkedHashMap**，accessOrder 参数传true，表示按照访问顺序排序，数据存储基于LinkedHashMap。

先看看LinkedHashMap 的原理吧

LinkedHashMap 继承 HashMap，在 HashMap 的基础上进行扩展，put 方法并没有重写，说明**LinkedHashMap遵循HashMap的数组加链表的结构**，

![img](https:////upload-images.jianshu.io/upload_images/11562793-c253143695c0734d?imageMogr2/auto-orient/strip|imageView2/2/w/500/format/webp)

LinkedHashMap重写了 **createEntry** 方法。

看下HashMap 的  `createEntry()` 方法

```csharp
void createEntry(int hash, K key, V value, int bucketIndex) {
    HashMapEntry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new HashMapEntry<>(hash, key, value, e);
    size++;
}
```

**HashMap的数组里面放的是`HashMapEntry` 对象**

看下LinkedHashMap 的  `createEntry()` 方法

```csharp
void createEntry(int hash, K key, V value, int bucketIndex) {
    HashMapEntry<K,V> old = table[bucketIndex];
    LinkedHashMapEntry<K,V> e = new LinkedHashMapEntry<>(hash, key, value, old);
    table[bucketIndex] = e; //数组的添加
    e.addBefore(header);  //处理链表
    size++;
}
```

**LinkedHashMap的数组里面放的是`LinkedHashMapEntry`对象**

**LinkedHashMapEntry**

```cpp
private static class LinkedHashMapEntry<K,V> extends HashMapEntry<K,V> {
    // These fields comprise the doubly linked list used for iteration.
    LinkedHashMapEntry<K,V> before, after; //双向链表

    private void remove() {
        before.after = after;
        after.before = before;
    }

    private void addBefore(LinkedHashMapEntry<K,V> existingEntry) {
        after  = existingEntry;
        before = existingEntry.before;
        before.after = this;
        after.before = this;
    }
```

**LinkedHashMapEntry继承 HashMapEntry，添加before和after变量，所以是一个双向链表结构，还添加了`addBefore`和`remove` 方法，用于新增和删除链表节点。**

**LinkedHashMapEntry#addBefore**  
 将一个数据添加到Header的前面

```cpp
private void addBefore(LinkedHashMapEntry<K,V> existingEntry) {
        after  = existingEntry;
        before = existingEntry.before;
        before.after = this;
        after.before = this;
}
```

existingEntry 传的都是链表头header，将一个节点添加到header节点前面，只需要移动链表指针即可，添加新数据都是放在链表头header 的before位置，**链表头节点header的before是最新访问的数据，header的after则是最旧的数据。**

再看下**LinkedHashMapEntry#remove**

```cpp
private void remove() {
        before.after = after;
        after.before = before;
    }
```

链表节点的移除比较简单，改变指针指向即可。

再看下LruCache的`put()`方法

```csharp
public final V put(K key, V value) {
    
    V previous;
    synchronized (this) {
        putCount++;
        //size增加
        size += safeSizeOf(key, value);
        // 1、linkHashMap的put方法
        previous = map.put(key, value);
        if (previous != null) {
            //如果有旧的值，会覆盖，所以大小要减掉
            size -= safeSizeOf(key, previous);
        }
    }

    trimToSize(maxSize);
    return previous;
}
```

LinkedHashMap 结构可以用这种图表示



![img](https:////upload-images.jianshu.io/upload_images/11562793-c37c29acfdcab2a6?imageMogr2/auto-orient/strip|imageView2/2/w/500/format/webp)

LinkedHashMap

LinkHashMap 的 put方法和get方法最后会调用`trimToSize`方法，**LruCache 重写`trimToSize`方法，判断内存如果超过一定大小，则移除最老的数据**

**LruCache#trimToSize，移除最老的数据**

```csharp
public void trimToSize(int maxSize) {
    while (true) {
        K key;
        V value;
        synchronized (this) {
            
            //大小没有超出，不处理
            if (size <= maxSize) {
                break;
            }

            //超出大小，移除最老的数据
            Map.Entry<K, V> toEvict = map.eldest();
            if (toEvict == null) {
                break;
            }

            key = toEvict.getKey();
            value = toEvict.getValue();
            map.remove(key);
            //这个大小的计算，safeSizeOf 默认返回1；
            size -= safeSizeOf(key, value);
            evictionCount++;
        }

        entryRemoved(true, key, value, null);
    }
}
```

对LinkHashMap 还不是很理解的话可以参考：
 [图解LinkedHashMap原理](https://www.jianshu.com/p/8f4f58b4b8ab)

LruCache小结：

- **LinkHashMap 继承HashMap，在 HashMap的基础上，新增了双向链表结构，每次访问数据的时候，会更新被访问的数据的链表指针，具体就是先在链表中删除该节点，然后添加到链表头header之前，这样就保证了链表头header节点之前的数据都是最近访问的（从链表中删除并不是真的删除数据，只是移动链表指针，数据本身在map中的位置是不变的）。**
- **LruCache 内部用LinkHashMap存取数据，在双向链表保证数据新旧顺序的前提下，设置一个最大内存，往里面put数据的时候，当数据达到最大内存的时候，将最老的数据移除掉，保证内存不超过设定的最大值。**

#### 2.3.2 磁盘缓存 DiskLruCache

依赖：

> implementation 'com.jakewharton:disklrucache:2.0.2'

DiskLruCache 跟 LruCache 实现思路是差不多的，一样是设置一个总大小，每次往硬盘写文件，总大小超过阈值，就会将旧的文件删除。简单看下remove操作：

```java
    // DiskLruCache 内部也是用LinkedHashMap
    private final LinkedHashMap<String, Entry> lruEntries =
        new LinkedHashMap<String, Entry>(0, 0.75f, true);
    ...

    public synchronized boolean remove(String key) throws IOException {
        checkNotClosed();
        validateKey(key);
        Entry entry = lruEntries.get(key);
        if (entry == null || entry.currentEditor != null) {
          return false;
        }
    
        //一个key可能对应多个value，hash冲突的情况
        for (int i = 0; i < valueCount; i++) {
          File file = entry.getCleanFile(i);
          //通过 file.delete() 删除缓存文件，删除失败则抛异常
          if (file.exists() && !file.delete()) {
            throw new IOException("failed to delete " + file);
          }
          size -= entry.lengths[i];
          entry.lengths[i] = 0;
        }
        ...
        return true;
  }
```

可以看到 DiskLruCache 同样是利用LinkHashMap的特点，只不过数组里面存的 Entry 有点变化，Editor 用于操作文件。

```java
private final class Entry {
    private final String key;

    private final long[] lengths;

    private boolean readable;

    private Editor currentEditor;

    private long sequenceNumber;
    ...
}
```

#### 2.4、防止OOM

加载图片非常重要的一点是需要防止OOM，上面的LruCache缓存大小设置，可以有效防止OOM，但是当图片需求比较大，可能需要设置一个比较大的缓存，这样的话发生OOM的概率就提高了，那应该探索其它防止OOM的方法。

##### 方法1：软引用

回顾一下Java的四大引用：

- 强引用： 普通变量都属于强引用，比如 `private Context context;`
- 软应用： SoftReference，在发生OOM之前，垃圾回收器会回收SoftReference引用的对象。
- 弱引用： WeakReference，发生GC的时候，垃圾回收器会回收WeakReference中的对象。
- 虚引用： 随时会被回收，没有使用场景。

怎么理解强引用：

> 强引用对象的回收时机依赖垃圾回收算法，我们常说的可达性分析算法，当Activity销毁的时候，Activity会跟GCRoot断开，至于GCRoot是谁？这里可以大胆猜想，Activity对象的创建是在ActivityThread中，ActivityThread要回调Activity的各个生命周期，肯定是持有Activity引用的，那么这个GCRoot可以认为就是ActivityThread，当Activity 执行onDestroy的时候，ActivityThread 就会断开跟这个Activity的联系，Activity到GCRoot不可达，所以会被垃圾回收器标记为可回收对象。

软引用的设计就是应用于会发生OOM的场景，大内存对象如Bitmap，可以通过 SoftReference 修饰，防止大对象造成OOM，看下这段代码

```csharp
    private static LruCache<String, SoftReference<Bitmap>> mLruCache = new LruCache<String, SoftReference<Bitmap>>(10 * 1024){
        @Override
        protected int sizeOf(String key, SoftReference<Bitmap> value) {
            //默认返回1，这里应该返回Bitmap占用的内存大小，单位：K

            //Bitmap被回收了，大小是0
            if (value.get() == null){
                return 0;
            }
            return value.get().getByteCount() /1024;
        }
    };
```

LruCache里存的是软引用对象，那么当内存不足的时候，Bitmap会被回收，也就是说通过SoftReference修饰的Bitmap就不会导致OOM。

当然，这段代码存在一些问题，Bitmap被回收的时候，LruCache剩余的大小应该重新计算，可以写个方法，当Bitmap取出来是空的时候，LruCache清理一下，重新计算剩余内存；

还有另一个问题，就是内存不足时软引用中的Bitmap被回收的时候，这个LruCache就形同虚设，相当于内存缓存失效了，必然出现效率问题。

##### 方法2：onLowMemory

**当内存不足的时候，Activity、Fragment会调用`onLowMemory`方法，可以在这个方法里去清除缓存，Glide使用的就是这一种方式来防止OOM。**

```cpp
//Glide
public void onLowMemory() {
    clearMemory();
}

public void clearMemory() {
    // Engine asserts this anyway when removing resources, fail faster and consistently
    Util.assertMainThread();
    // memory cache needs to be cleared before bitmap pool to clear re-pooled Bitmaps too. See #687.
    memoryCache.clearMemory();
    bitmapPool.clearMemory();
    arrayPool.clearMemory();
  }
```

##### 方法3：从Bitmap 像素存储位置考虑

我们知道，系统为每个进程，也就是每个虚拟机分配的内存是有限的，早期的16M、32M，现在100+M，
 虚拟机的内存划分主要有5部分：

- 虚拟机栈
- 本地方法栈
- 程序计数器
- 方法区
- 堆

而对象的分配一般都是在堆中，堆是JVM中最大的一块内存，OOM一般都是发生在堆中。

Bitmap 之所以占内存大不是因为对象本身大，而是因为Bitmap的像素数据，
 **Bitmap的像素数据大小 = 宽 \* 高 \* 1像素占用的内存。**

1像素占用的内存是多少？不同格式的Bitmap对应的像素占用内存是不同的，具体是多少呢？
 在Fresco中看到如下定义代码

```java
  /**
   * Bytes per pixel definitions
   */
  public static final int ALPHA_8_BYTES_PER_PIXEL = 1;
  public static final int ARGB_4444_BYTES_PER_PIXEL = 2;
  public static final int ARGB_8888_BYTES_PER_PIXEL = 4;
  public static final int RGB_565_BYTES_PER_PIXEL = 2;
  public static final int RGBA_F16_BYTES_PER_PIXEL = 8;
```

如果Bitmap使用 `RGB_565` 格式，则1像素占用 2 byte，`ARGB_8888` 格式则占4 byte。
 **在选择图片加载框架的时候，可以将内存占用这一方面考虑进去，更少的内存占用意味着发生OOM的概率越低。** Glide内存开销是Picasso的一半，就是因为默认Bitmap格式不同。

至于宽高，是指Bitmap的宽高，怎么计算的呢？看`BitmapFactory.Options` 的 outWidth

```dart
/**
     * The resulting width of the bitmap. If {@link #inJustDecodeBounds} is
     * set to false, this will be width of the output bitmap after any
     * scaling is applied. If true, it will be the width of the input image
     * without any accounting for scaling.
     *
     * <p>outWidth will be set to -1 if there is an error trying to decode.</p>
     */
    public int outWidth;
```

看注释的意思，如果 `BitmapFactory.Options` 中指定 `inJustDecodeBounds` 为true，则为原图宽高，如果是false，则是缩放后的宽高。**所以我们一般可以通过压缩来减小Bitmap像素占用内存**。

扯远了，上面分析了Bitmap像素数据大小的计算，只是说明Bitmap像素数据为什么那么大。**那是否可以让像素数据不放在java堆中，而是放在native堆中呢**？据说Android 3.0到8.0 之间Bitmap像素数据存在Java堆，而8.0之后像素数据存到native堆中，是不是真的？看下源码就知道了~

###### 8.0 Bitmap

java层创建Bitmap方法

```dart
    public static Bitmap createBitmap(@Nullable DisplayMetrics display, int width, int height,
            @NonNull Config config, boolean hasAlpha, @NonNull ColorSpace colorSpace) {
        ...
        Bitmap bm;
        ...
        if (config != Config.ARGB_8888 || colorSpace == ColorSpace.get(ColorSpace.Named.SRGB)) {
            //最终都是通过native方法创建
            bm = nativeCreate(null, 0, width, width, height, config.nativeInt, true, null, null);
        } else {
            bm = nativeCreate(null, 0, width, width, height, config.nativeInt, true,
                    d50.getTransform(), parameters);
        }

        ...
        return bm;
    }
```

Bitmap 的创建是通过native方法 `nativeCreate`

对应源码
 [8.0.0_r4/xref/frameworks/base/core/jni/android/graphics/Bitmap.cpp](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.androidos.net.cn%2Fandroid%2F8.0.0_r4%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjni%2Fandroid%2Fgraphics%2FBitmap.cpp)

```dart
//Bitmap.cpp
static const JNINativeMethod gBitmapMethods[] = {
    {   "nativeCreate",             "([IIIIIIZ[FLandroid/graphics/ColorSpace$Rgb$TransferParameters;)Landroid/graphics/Bitmap;",
        (void*)Bitmap_creator },
...
```

JNI动态注册，nativeCreate 方法 对应 `Bitmap_creator`；

```cpp
//Bitmap.cpp
static jobject Bitmap_creator(JNIEnv* env, jobject, jintArray jColors,
                              jint offset, jint stride, jint width, jint height,
                              jint configHandle, jboolean isMutable,
                              jfloatArray xyzD50, jobject transferParameters) {
    ...
    //1. 申请堆内存，创建native层Bitmap
    sk_sp<Bitmap> nativeBitmap = Bitmap::allocateHeapBitmap(&bitmap, NULL);
    if (!nativeBitmap) {
        return NULL;
    }

    ...
    //2.创建java层Bitmap
    return createBitmap(env, nativeBitmap.release(), getPremulBitmapCreateFlags(isMutable));
}
```

主要两个步骤：

1. 申请内存，创建native层Bitmap，看下`allocateHeapBitmap`方法
    [8.0.0_r4/xref/frameworks/base/libs/hwui/hwui/Bitmap.cpp](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.androidos.net.cn%2Fandroid%2F8.0.0_r4%2Fxref%2Fframeworks%2Fbase%2Flibs%2Fhwui%2Fhwui%2FBitmap.cpp)

```cpp
//
static sk_sp<Bitmap> allocateHeapBitmap(size_t size, const SkImageInfo& info, size_t rowBytes,
        SkColorTable* ctable) {
    // calloc 是c++ 的申请内存函数
    void* addr = calloc(size, 1);
    if (!addr) {
        return nullptr;
    }
    return sk_sp<Bitmap>(new Bitmap(addr, size, info, rowBytes, ctable));
}
```

可以看到通过c++的 `calloc` 函数申请了一块内存空间，然后创建native层Bitmap对象，把内存地址传过去，也就是native层的Bitmap数据（像素数据）是存在native堆中。

1. 创建java 层Bitmap

```cpp
//Bitmap.cpp
jobject createBitmap(JNIEnv* env, Bitmap* bitmap,
        int bitmapCreateFlags, jbyteArray ninePatchChunk, jobject ninePatchInsets,
        int density) {
    ...
    BitmapWrapper* bitmapWrapper = new BitmapWrapper(bitmap);
     //通过JNI回调Java层，调用java层的Bitmap构造方法
    jobject obj = env->NewObject(gBitmap_class, gBitmap_constructorMethodID,
            reinterpret_cast<jlong>(bitmapWrapper), bitmap->width(), bitmap->height(), density,
            isMutable, isPremultiplied, ninePatchChunk, ninePatchInsets);

   ...
    return obj;
}
```

env->NewObject，通过JNI创建Java层Bitmap对象，`gBitmap_class，gBitmap_constructorMethodID`这些变量是什么意思，看下面这个方法，对应java层的Bitmap的类名和构造方法。

```cpp
//Bitmap.cpp
int register_android_graphics_Bitmap(JNIEnv* env)
{
    gBitmap_class = MakeGlobalRefOrDie(env, FindClassOrDie(env, "android/graphics/Bitmap"));
    gBitmap_nativePtr = GetFieldIDOrDie(env, gBitmap_class, "mNativePtr", "J");
    gBitmap_constructorMethodID = GetMethodIDOrDie(env, gBitmap_class, "<init>", "(JIIIZZ[BLandroid/graphics/NinePatch$InsetStruct;)V");
    gBitmap_reinitMethodID = GetMethodIDOrDie(env, gBitmap_class, "reinit", "(IIZ)V");
    gBitmap_getAllocationByteCountMethodID = GetMethodIDOrDie(env, gBitmap_class, "getAllocationByteCount", "()I");
    return android::RegisterMethodsOrDie(env, "android/graphics/Bitmap", gBitmapMethods,
                                         NELEM(gBitmapMethods));
}
```

8.0 的Bitmap创建就两个点：

1. 创建native层Bitmap，在native堆申请内存。
2. 通过JNI创建java层Bitmap对象，这个对象在java堆中分配内存。

像素数据是存在native层Bitmap，也就是证明8.0的Bitmap像素数据存在native堆中。

###### 7.0 Bitmap

直接看native层的方法，

[/7.0.0_r31/xref/frameworks/base/core/jni/android/graphics/Bitmap.cpp](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.androidos.net.cn%2Fandroid%2F7.0.0_r31%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjni%2Fandroid%2Fgraphics%2FBitmap.cpp)

```php
//JNI动态注册
static const JNINativeMethod gBitmapMethods[] = {
    {   "nativeCreate",             "([IIIIIIZ)Landroid/graphics/Bitmap;",
        (void*)Bitmap_creator },
...

static jobject Bitmap_creator(JNIEnv* env, jobject, jintArray jColors,
                              jint offset, jint stride, jint width, jint height,
                              jint configHandle, jboolean isMutable) {
    ... 
    //1.通过这个方法来创建native层Bitmap
    Bitmap* nativeBitmap = GraphicsJNI::allocateJavaPixelRef(env, &bitmap, NULL);
    ...

    return GraphicsJNI::createBitmap(env, nativeBitmap,
            getPremulBitmapCreateFlags(isMutable));
}
```

native层Bitmap 创建是通过`GraphicsJNI::allocateJavaPixelRef`，看看里面是怎么分配的，
 GraphicsJNI 的实现类是[Graphics.cpp](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.androidos.net.cn%2Fandroid%2F7.0.0_r31%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjni%2Fandroid%2Fgraphics%2FGraphics.cpp)

```php
android::Bitmap* GraphicsJNI::allocateJavaPixelRef(JNIEnv* env, SkBitmap* bitmap,
                                             SkColorTable* ctable) {
    const SkImageInfo& info = bitmap->info();
    
    size_t size;
    //计算需要的空间大小
    if (!computeAllocationSize(*bitmap, &size)) {
        return NULL;
    }

    // we must respect the rowBytes value already set on the bitmap instead of
    // attempting to compute our own.
    const size_t rowBytes = bitmap->rowBytes();
    // 1. 创建一个数组，通过JNI在java层创建的
    jbyteArray arrayObj = (jbyteArray) env->CallObjectMethod(gVMRuntime,
                                                             gVMRuntime_newNonMovableArray,
                                                             gByte_class, size);
    ...
    // 2. 获取创建的数组的地址
    jbyte* addr = (jbyte*) env->CallLongMethod(gVMRuntime, gVMRuntime_addressOf, arrayObj);
    ...
    //3. 创建Bitmap，传这个地址
    android::Bitmap* wrapper = new android::Bitmap(env, arrayObj, (void*) addr,
            info, rowBytes, ctable);
    wrapper->getSkBitmap(bitmap);
    // since we're already allocated, we lockPixels right away
    // HeapAllocator behaves this way too
    bitmap->lockPixels();

    return wrapper;
}
```

可以看到，7.0 像素内存的分配是这样的：

1. 通过JNI调用java层创建一个数组
2. 然后创建native层Bitmap，把数组的地址传进去。

由此说明，7.0 的Bitmap像素数据是放在java堆的。

当然，3.0 以下Bitmap像素内存据说也是放在native堆的，但是需要手动释放native层的Bitmap，也就是需要手动调用recycle方法，native层内存才会被回收。这个大家可以自己去看源码验证。

###### native层Bitmap 回收问题

Java层的Bitmap对象由垃圾回收器自动回收，而native层Bitmap印象中我们是不需要手动回收的，源码中如何处理的呢？

记得有个面试题是这样的：

> 说说final、finally、finalize 的关系

三者除了长得像，其实没有半毛钱关系，final、finally大家都用的比较多，而 `finalize` 用的少，或者没用过，`finalize` 是 Object 类的一个方法，注释是这样的：

```dart
/**
     * Called by the garbage collector on an object when garbage collection
     * determines that there are no more references to the object.
     * A subclass overrides the {@code finalize} method to dispose of
     * system resources or to perform other cleanup.
     * <p>
     ...**/
  protected void finalize() throws Throwable { }
```

意思是说，垃圾回收器确认这个对象没有其它地方引用到它的时候，会调用这个对象的`finalize`方法，子类可以重写这个方法，做一些释放资源的操作。

**在6.0以前，Bitmap 就是通过这个finalize 方法来释放native层对象的。**
 [6.0 Bitmap.java](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.androidos.net.cn%2Fandroid%2F6.0.1_r16%2Fxref%2Fframeworks%2Fbase%2Fgraphics%2Fjava%2Fandroid%2Fgraphics%2FBitmap.java)

```java
Bitmap(long nativeBitmap, byte[] buffer, int width, int height, int density,
            boolean isMutable, boolean requestPremultiplied,
            byte[] ninePatchChunk, NinePatch.InsetStruct ninePatchInsets) {
        ...
        mNativePtr = nativeBitmap;
        //1.创建 BitmapFinalizer
        mFinalizer = new BitmapFinalizer(nativeBitmap);
        int nativeAllocationByteCount = (buffer == null ? getByteCount() : 0);
        mFinalizer.setNativeAllocationByteCount(nativeAllocationByteCount);
}

 private static class BitmapFinalizer {
        private long mNativeBitmap;

        // Native memory allocated for the duration of the Bitmap,
        // if pixel data allocated into native memory, instead of java byte[]
        private int mNativeAllocationByteCount;

        BitmapFinalizer(long nativeBitmap) {
            mNativeBitmap = nativeBitmap;
        }

        public void setNativeAllocationByteCount(int nativeByteCount) {
            if (mNativeAllocationByteCount != 0) {
                VMRuntime.getRuntime().registerNativeFree(mNativeAllocationByteCount);
            }
            mNativeAllocationByteCount = nativeByteCount;
            if (mNativeAllocationByteCount != 0) {
                VMRuntime.getRuntime().registerNativeAllocation(mNativeAllocationByteCount);
            }
        }

        @Override
        public void finalize() {
            try {
                super.finalize();
            } catch (Throwable t) {
                // Ignore
            } finally {
                //2.就是这里了，
                setNativeAllocationByteCount(0);
                nativeDestructor(mNativeBitmap);
                mNativeBitmap = 0;
            }
        }
    }
```

在Bitmap构造方法创建了一个 `BitmapFinalizer`类，重写finalize 方法，在java层Bitmap被回收的时候，BitmapFinalizer 对象也会被回收，finalize 方法肯定会被调用，在里面释放native层Bitmap对象。

6.0 之后做了一些变化，BitmapFinalizer 没有了，被[NativeAllocationRegistry](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.androidos.net.cn%2Fandroid%2F8.0.0_r4%2Fxref%2Flibcore%2Fluni%2Fsrc%2Fmain%2Fjava%2Flibcore%2Futil%2FNativeAllocationRegistry.java)取代。

例如 8.0 Bitmap构造方法

```java
    Bitmap(long nativeBitmap, int width, int height, int density,
            boolean isMutable, boolean requestPremultiplied,
            byte[] ninePatchChunk, NinePatch.InsetStruct ninePatchInsets) {
       
        ...
        mNativePtr = nativeBitmap;
        long nativeSize = NATIVE_ALLOCATION_SIZE + getAllocationByteCount();
        //  创建NativeAllocationRegistry这个类，调用registerNativeAllocation 方法
        NativeAllocationRegistry registry = new NativeAllocationRegistry(
            Bitmap.class.getClassLoader(), nativeGetNativeFinalizer(), nativeSize);
        registry.registerNativeAllocation(this, nativeBitmap);
    }
```

NativeAllocationRegistry 就不分析了，
 **不管是BitmapFinalizer 还是NativeAllocationRegistry，目的都是在java层Bitmap被回收的时候，将native层Bitmap对象也回收掉。** 一般情况下我们无需手动调用recycle方法，由GC去盘它即可。

上面分析了Bitmap像素存储位置，我们知道，Android 8.0 之后Bitmap像素内存放在native堆，Bitmap导致OOM的问题基本不会在8.0以上设备出现了（没有内存泄漏的情况下），那8.0 以下设备怎么办？赶紧升级或换手机吧~

![img](https:////upload-images.jianshu.io/upload_images/11562793-347ede94c553a0b8?imageMogr2/auto-orient/strip|imageView2/2/w/240/format/webp)

image

我们换手机当然没问题，但是并不是所有人都能跟上Android系统更新的步伐，所以，问题还是要解决~

Fresco 之所以能跟Glide 正面交锋，必然有其独特之处，文中开头列出 Fresco 的优点是：“在5.0以下(最低2.3)系统，Fresco将图片放到一个特别的内存区域(Ashmem区)”
 这个Ashmem区是一块匿名共享内存，Fresco 将Bitmap像素放到共享内存去了，共享内存是属于native堆内存。

Fresco 关键源码在 `PlatformDecoderFactory` 这个类

```java
public class PlatformDecoderFactory {

  /**
   * Provide the implementation of the PlatformDecoder for the current platform using the provided
   * PoolFactory
   *
   * @param poolFactory The PoolFactory
   * @return The PlatformDecoder implementation
   */
  public static PlatformDecoder buildPlatformDecoder(
      PoolFactory poolFactory, boolean gingerbreadDecoderEnabled) {
    //8.0 以上用 OreoDecoder 这个解码器
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
      int maxNumThreads = poolFactory.getFlexByteArrayPoolMaxNumThreads();
      return new OreoDecoder(
          poolFactory.getBitmapPool(), maxNumThreads, new Pools.SynchronizedPool<>(maxNumThreads));
    } else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
      //大于5.0小于8.0用 ArtDecoder 解码器
      int maxNumThreads = poolFactory.getFlexByteArrayPoolMaxNumThreads();
      return new ArtDecoder(
          poolFactory.getBitmapPool(), maxNumThreads, new Pools.SynchronizedPool<>(maxNumThreads));
    } else {
      if (gingerbreadDecoderEnabled && Build.VERSION.SDK_INT < Build.VERSION_CODES.KITKAT) {
        //小于4.4 用 GingerbreadPurgeableDecoder 解码器
        return new GingerbreadPurgeableDecoder();
      } else {
        //这个就是4.4到5.0 用的解码器了
        return new KitKatPurgeableDecoder(poolFactory.getFlexByteArrayPool());
      }
    }
  }
}
```

8.0 先不看了，看一下 4.4 以下是怎么得到Bitmap的，看下`GingerbreadPurgeableDecoder`这个类有个获取Bitmap的方法

```csharp
//GingerbreadPurgeableDecoder
private Bitmap decodeFileDescriptorAsPurgeable(
      CloseableReference<PooledByteBuffer> bytesRef,
      int inputLength,
      byte[] suffix,
      BitmapFactory.Options options) {
    //  MemoryFile ：匿名共享内存
    MemoryFile memoryFile = null;
    try {
      //将图片数据拷贝到匿名共享内存
      memoryFile = copyToMemoryFile(bytesRef, inputLength, suffix);
      FileDescriptor fd = getMemoryFileDescriptor(memoryFile);
      if (mWebpBitmapFactory != null) {
        // 创建Bitmap，Fresco自己写了一套创建Bitmap方法
        Bitmap bitmap = mWebpBitmapFactory.decodeFileDescriptor(fd, null, options);
        return Preconditions.checkNotNull(bitmap, "BitmapFactory returned null");
      } else {
        throw new IllegalStateException("WebpBitmapFactory is null");
      }
    } 
  }
```

捋一捋，4.4以下，Fresco 使用匿名共享内存来保存Bitmap数据，首先将图片数据拷贝到匿名共享内存中，然后使用Fresco自己写的加载Bitmap的方法。

Fresco对不同Android版本使用不同的方式去加载Bitmap，至于4.4-5.0，5.0-8.0，8.0 以上，对应另外三个解码器，大家可以从`PlatformDecoderFactory` 这个类入手，自己去分析，思考为什么不同平台要分这么多个解码器，8.0 以下都用匿名共享内存不好吗？期待你在评论区跟大家分享~

### 2.5 ImageView 内存泄露

> 曾经在Vivo驻场开发，带有头像功能的页面被测出内存泄漏，原因是SDK中有个加载网络头像的方法，持有ImageView引用导致的。

当然，修改也比较简单粗暴，**将ImageView用WeakReference修饰**就完事了。

事实上，这种方式虽然解决了内存泄露问题，但是并不完美，例如在界面退出的时候，我们除了希望ImageView被回收，同时希望加载图片的任务可以取消，队未执行的任务可以移除。

Glide的做法是监听生命周期回调，看 `RequestManager` 这个类

```cpp
public void onDestroy() {
    targetTracker.onDestroy();
    for (Target<?> target : targetTracker.getAll()) {
      //清理任务
      clear(target);
    }
    targetTracker.clear();
    requestTracker.clearRequests();
    lifecycle.removeListener(this);
    lifecycle.removeListener(connectivityMonitor);
    mainHandler.removeCallbacks(addSelfToLifecycle);
    glide.unregisterRequestManager(this);
  }
```

在Activity/fragment 销毁的时候，取消图片加载任务，细节大家可以自己去看源码。

### 2.6 列表加载问题

#### 图片错乱

由于RecyclerView或者LIstView的复用机制，网络加载图片开始的时候ImageView是第一个item的，加载成功之后ImageView由于复用可能跑到第10个item去了，在第10个item显示第一个item的图片肯定是错的。

常规的做法是给ImageView设置tag，tag一般是图片地址，更新ImageView之前判断tag是否跟url一致。

当然，可以在item从列表消失的时候，取消对应的图片加载任务。要考虑放在图片加载框架做还是放在UI做比较合适。

#### 线程池任务过多

列表滑动，会有很多图片请求，如果是第一次进入，没有缓存，那么队列会有很多任务在等待。所以在请求网络图片之前，需要判断队列中是否已经存在该任务，存在则不加到队列去。

## 总结

本文通过Glide开题，分析一个图片加载框架必要的需求，以及各个需求涉及到哪些技术和原理。

- 异步加载：最少两个线程池
- 切换到主线程：Handler
- 缓存：LruCache、DiskLruCache，涉及到LinkHashMap原理
- 防止OOM：软引用、LruCache、图片压缩没展开讲、Bitmap像素存储位置源码分析、Fresco部分源码分析
- 内存泄露：注意ImageView的正确引用，生命周期管理
- 列表滑动加载的问题：加载错乱用tag、队满任务存在则不添加

文中也遗留一些问题，例如：
 Fresco为什么要在不同Android版本上使用不同解码器去获取Bitmap，8.0以下都用匿名共享内存不可以吗？期待你主动学习并且跟大家分享~

------

就这样，有问题评论区留言~

相关文章：
 [图解LinkedHashMap原理](https://www.jianshu.com/p/8f4f58b4b8ab)
 [谈谈fresco的bitmap内存分配](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fchiefhsing%2Farticle%2Fdetails%2F53899242)



作者：蓝师傅_Android
链接：https://www.jianshu.com/p/1ab5597af607
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。