LeakCanary是使用成本较低的HeapProfiler, 通常内存泄漏都比较隐蔽,  和OOM后再去分析hprof文件不同,他能在开发过程中帮助我们及时发现可能泄露的问题.
# 原理
LeakCanary的原理很简单: 在Activity或Fragment被销毁后, 将他们的引用包装成一个`WeakReference`, 然后将这个`WeakReference`关联到一个`ReferenceQueue`.查看`ReferenceQueue`中是否含有Activity或Fragment的引用, 如果没有触发GC,再次查看,还是没有的话就说明没有回收成功, 可能发生了泄露. 这时候开始dump内存的信息,并分析泄露的引用链.

#使用
LeakCanary现在已经有2.0的canary版本.实现和上个版本1.6相比也有不同.
2.0以前

```java
"leakcanary"          : 'com.squareup.leakcanary:leakcanary-android:1.5',
"leakcanary_no"       : 'com.squareup.leakcanary:leakcanary-android-no-op:1.5'

public class MyApp extends MultiDexApplication {
    @Override
    public void onCreate() {
        super.onCreate();
        //2.0之前LeakCanary是在单独的进程里来分析的
        if (LeakCanary.isInAnalyzerProcess(this)) {
          return;
        }
        LeakCanary.install(this);
    }
}
```
2.0

```kotlin
debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.0-alpha-2'
```
区别
* 配合Gradle3.0引入的`debugImplementation`我们不需要添加no-op包来分离测试和生产环境了. 
* LeakCanary也不需要再单独的进程里来分析了,2.0之前的版本AS里可以看到LeakCanary是单独开了一个进程的
* 2.0不再需要我们手动执行初始化了, 我们看看2.0自动注册的实现.

#### 自动注册的实现
查看leakCanary-android  ->引用了 leakcanary-android-core →引用了leakcanary-leaksentry
查看leakcanary-leaksentry的AndroidManifest.xml文件

```kotlin
<?xml version="1.0" encoding="utf-8"?>
<manifest
    xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.squareup.leakcanary.leaksentry"
    >

  <application>
    <provider
        android:name="leakcanary.internal.LeakSentryInstaller"
        android:authorities="${applicationId}.leak-sentry-installer"
        android:exported="false"/>
  </application>
</manifest>
```
LeakSentryInstaller继承了ContentProvider, 在onCreate()方法里调用了`InternalLeakSentry.install(application)`
```kotlin
/**
 * Content providers are loaded before the application class is created. [LeakSentryInstaller] is
 * used to install [leaksentry.LeakSentry] on application start.
 */
internal class LeakSentryInstaller : ContentProvider() {

  override fun onCreate(): Boolean {
    CanaryLog.logger = DefaultCanaryLog()
    val application = context!!.applicationContext as Application
    //在contentProvider初始化的时候, 初始化LeakCanary
    InternalLeakSentry.install(application)
    return true
  }

  override fun query(
    uri: Uri,
    strings: Array<String>?,
    s: String?,
    strings1: Array<String>?,
    s1: String?
  ): Cursor? {
    return null
  }

  override fun getType(uri: Uri): String? {
    return null
  }

  override fun insert(
    uri: Uri,
    contentValues: ContentValues?
  ): Uri? {
    return null
  }

  override fun delete(
    uri: Uri,
    s: String?,
    strings: Array<String>?
  ): Int {
    return 0
  }

  override fun update(
    uri: Uri,
    contentValues: ContentValues?,
    s: String?,
    strings: Array<String>?
  ): Int {
    return 0
  }
}
```
APP构建经过manifest-merge后[官方文档-合并多个清单文件](https://developer.android.com/studio/build/manifest-merge?hl=zh-cn),这个ContentProvider会被合并到唯一的manifest.xml中.  当APP初始化时会加载这个`LeakSentryInstaller`,就会自动帮我们执行`InternalLeakSentry.install(application)`

**作为ContentProvider他的其他CRUD实现都是空的.作者只是巧妙利用了ContentProvider无需显式初始化的特性(对比`Service,BroadcastReceiver`)来实现了自动注册.**

# 源码分析
## 1、InternalLeakSentry#install
这里初始化了RefWatcher, 
```kotlin
internal object InternalLeakSentry {
  private val checkRetainedExecutor = Executor {
    //这里的Executor收到任务后, 延迟watchDurationMillis秒后执行,这里默认是5秒
    mainHandler.postDelayed(it, LeakSentry.config.watchDurationMillis)
  }
  //初始化RefWatcher
  val refWatcher = RefWatcher(
      clock = clock,
      checkRetainedExecutor = checkRetainedExecutor,
      onReferenceRetained = { listener.onReferenceRetained() },
      isEnabled = { LeakSentry.config.enabled } //默认true
  )
  
  //初始化LeakSentryListener
  private val listener: LeakSentryListener
  init {
    listener = try {
      //反射得到LeakSentryListener实现类: InternalLeakCanary实例
      val leakCanaryListener = Class.forName("leakcanary.internal.InternalLeakCanary")
      leakCanaryListener.getDeclaredField("INSTANCE").get(null) as LeakSentryListener
    } catch (ignored: Throwable) {
      LeakSentryListener.None
    }
  }

  fun install(application: Application) {
    CanaryLog.d("Installing LeakSentry")
    //需要在主线程初始化
    checkMainThread()  
    //避免重复初始化
    if (this::application.isInitialized) {
      return
    }
    InternalLeakSentry.application = application
    //可以通过LeakSentry.config来修改watchFragments, watchDurationMillis等参数
    val configProvider = { LeakSentry.config }
    //注册ActivityDestroyWatcher
    ActivityDestroyWatcher.install(
        application, refWatcher, configProvider
    )
    //注册FragmentDestroyWatcher
    FragmentDestroyWatcher.install(
        application, refWatcher, configProvider
    )
    //LeakSentry初始化完成回调
    listener.onLeakSentryInstalled(application)
  }
}
```
#### 1.1 初始化RefWatcher
RefWatcher的参数`checkRetainedExecutor` 和 `onReferenceRetained`

* 在`checkRetainedExecutor`里, 会把任务`mainHandler.postDelayed(it, LeakSentry.config.watchDurationMillis)`默认延迟5秒执行.
* `onReferenceRetained = { listener.onReferenceRetained() }`是Kotlin中的高阶函数, 这里的listener是 InternalLeakCanary ;

#### 1.2 ActivityDestroyWatcher#install()
看名字很容易理解, 这是用来观测Activity.onDestroy()的,同理FragmentDestroyWatcher是观测Fragment.onDestroy()的,当然2.0还是保留了watch方法传入我们想要观测的类型.

内存泄露的定义就是超出了预期的生命周期没有被回收. 对于安卓来说, Activity和Fragment 因为都有明确的生命周期onCreate -> onDestroy, 可以视作最容易被观测的颗粒度.


####  1.2 LeakSentryListener#onLeakSentryInstalled(application)
这里是LeakSentry初始化结束的回调, 查看 LeakSentryListener 的实现类  InternalLeakCanary.
```kotlin
internal object InternalLeakCanary : leakcanary.internal.LeakSentryListener {
  override fun onLeakSentryInstalled(application: Application) {
    this.application = application
    // 创建HeapDumper
    val heapDumper = AndroidHeapDumper(application, leakDirectoryProvider)
    // 用于在通过RefrenceQueue判断内存泄露之前,手动触发一次GC 
    val gcTrigger = GcTrigger.Default
    //和LeakSentry一样, LeakCanary也支持配置
    val configProvider = { LeakCanary.config }
    //创建HandlerThread来执行耗时操作
    val handlerThread = HandlerThread(HeapDumpTrigger.LEAK_CANARY_THREAD_NAME)
    handlerThread.start()
    val backgroundHandler = Handler(handlerThread.looper)
    // HeadDump触发器
    heapDumpTrigger = HeapDumpTrigger(
        application, backgroundHandler, LeakSentry.refWatcher, gcTrigger, heapDumper, configProvider
    )
    //这里是kotlin的扩展方法, 下面会分析
    application.registerVisibilityListener { applicationVisible ->
      this.applicationVisible = applicationVisible
      heapDumpTrigger.onApplicationVisibilityChanged(applicationVisible)
    }
    addDynamicShortcut(application)
  }
}
```
在onLeakSentryInstalled()里做了以下操作:
1. 初始化`gcTrigger`用来触发GC.
2. 新建了一个HandlerThread用来异步处理headpDump.
3. 初始化`heapDumpTrigger`.  用来生成heapDump文件. 
4. application.registerVisibilityListener()
这里我们要看一下 `application.registerVisibilityListener()`, Application是没有这个方法的, 他使用了Kotlin的扩展函数
```kotlin
internal class VisibilityTracker(private val listener: (Boolean) -> Unit
) : ActivityLifecycleCallbacksAdapter() {

    private var startedActivityCount = 0

private var hasVisibleActivities: Boolean = false
  // 打开新页面, count++
  override fun onActivityStarted(activity: Activity) {
    startedActivityCount++
    if (!hasVisibleActivities && startedActivityCount == 1) {
      hasVisibleActivities = true
      listener.invoke(true)
    }
  }
  //页面stop, count--
  override fun onActivityStopped(activity: Activity) {
    if (startedActivityCount > 0) {
      startedActivityCount--
    }
    //startedActivityCount == 0, APP退到后台的情况.
    if (hasVisibleActivities && startedActivityCount == 0 && !activity.isChangingConfigurations) {
      hasVisibleActivities = false
      listener.invoke(false)
    }
  }
}

//Application扩展的方法在这
internal fun Application.registerVisibilityListener(listener: (Boolean) -> Unit) {
  //观测Activity生命周期
  registerActivityLifecycleCallbacks(VisibilityTracker(listener))
}
```
VisibilityTracker内重写了两个方法, `onActivityStarted()`和`onActivityStopped()`, 逻辑很简单, 页面onStart()时count++, onStop()时count--. 
只有APP退到后台的情况, 满足条件调用`listener.invoke(false)`.

回到
```kotlin
    application.registerVisibilityListener { applicationVisible ->
      this.applicationVisible = applicationVisible
      heapDumpTrigger.onApplicationVisibilityChanged(applicationVisible)
    }
```
heapDumpTrigger.onApplicationVisibilityChanged(applicationVisible)
```kotlin
  fun onApplicationVisibilityChanged(applicationVisible: Boolean) {
    if (applicationVisible) {
      applicationInvisibleAt = -1L
    } else {
      //App退到后台的情况
      applicationInvisibleAt = SystemClock.uptimeMillis()
      scheduleRetainedInstanceCheck("app became invisible", LeakSentry.config.watchDurationMillis)
    }
  }
  
  //backgroundHandler延迟5秒执行checkRetainedInstances()
  private fun scheduleRetainedInstanceCheck(
    reason: String,
    delayMillis: Long
  ) {
    backgroundHandler.postDelayed({
      //检查残留的实例, 下面会分析
      checkRetainedInstances(reason)
    }, delayMillis)
  }
```
总结一下, 在APP退到后台5秒后进行检查泄露的任务.

## 2、ActivityDestroyWatcher#install
伴生对象方法hinstall()里创建了ActivityDestroyWatcher, 
 调用`application.registerActivityLifecycleCallbacks()`来检测Activity的生命周期, 在自定义的 lifecycleCallbacks 的`onActivityDestroyed()`时调用`refWatcher.watch(activity)`来观察Activity的引用.
```kotlin
internal class ActivityDestroyWatcher private constructor(
  private val refWatcher: RefWatcher,
  private val configProvider: () -> Config
) {
  private val lifecycleCallbacks = object : ActivityLifecycleCallbacksAdapter() {
    //Activity onDestroy的回调
    override fun onActivityDestroyed(activity: Activity) {
      if (configProvider().watchActivities) {
        refWatcher.watch(activity)
      }
    }
  }

  companion object {
    fun install(
      application: Application,
      refWatcher: RefWatcher,
      configProvider: () -> Config
    ) {
      //Application注册检测Activity声明周期
      val activityDestroyWatcher = ActivityDestroyWatcher(refWatcher, configProvider)
      application.registerActivityLifecycleCallbacks(activityDestroyWatcher.lifecycleCallbacks)
    }
  }
}
```

## 3、RefWatcher.watch(activity)
RefWatcher在第一步已经完成了初始化, 开始看watch()代码之前先熟悉下RefWatcher的三个对象`watchedReferences`,`retainedReferences `,`queue`
```kotlin
  //等待被观察的对象, 还未被移动到retainedReferences列表
  private val watchedReferences = mutableMapOf<String, KeyedWeakReference>()
 
  // 超出了预期的生命周期, 我们觉得可能会发生泄露的对象
  private val retainedReferences = mutableMapOf<String, KeyedWeakReference>()

  // ReferenceQueue用于判断弱引用所持有的对象是否已被GC
  private val queue = ReferenceQueue<Any>()
```
开始看watch()代码
```kotlin
class RefWatcher {
  //watchedReference可以是Activity,Fragment,或者我们传入的任意对象
  @Synchronized fun watch(watchedReference: Any) {
    watch(watchedReference, "")
  }
 
  @Synchronized fun watch(
    watchedReference: Any,  
    referenceName: String
  ) {
    if (!isEnabled()) {
      return
    }
    //删除弱可达的引用
    removeWeaklyReachableReferences()
    //给每个watchedReference生成唯一的key
    val key = UUID.randomUUID().toString()
    val watchUptimeMillis = clock.uptimeMillis()
    //生成watchedReference的一个弱引用
    val reference =KeyedWeakReference(watchedReference, key, referenceName, watchUptimeMillis, queue)
    if (referenceName != "") {
      CanaryLog.d(
          "Watching instance of %s named %s with key %s", reference.className,
          referenceName, key
      )
    } else {
      CanaryLog.d(
          "Watching instance of %s with key %s", reference.className, key
      )
    }
    //加入watchedReferences(LinkedHashMap)
    watchedReferences[key] = reference
    //交给executor执行, 看第一步可以知道延迟5秒后执行
    checkRetainedExecutor.execute {
      moveToRetained(key) //移动到retainedReferences
    }
  }

  @Synchronized private fun moveToRetained(key: String) {
    removeWeaklyReachableReferences()
    //从watchedReferences中移除这个弱引用元素
    val retainedRef = watchedReferences.remove(key)
    //添加到 retainedReferences
    if (retainedRef != null) {
      retainedReferences[key] = retainedRef
      onReferenceRetained()
    }
  }
}
```

1. 通过UUID生成这个被观测对象`watchedReference`(Activity,Fragment...)对应的key, 根据`watchedReference`创建弱引用`KeyedWeakReference`作为value ,加入LinkedHashMap `watchedReferences`中. 键值对<UUID, KeyedWeakReference>     
2. 使用`checkRetainedExecutor.execute {
      moveToRetained(key) 
    }`调度, 看第一步可以知道5秒后执行`moveToRetained()`任务
3. 执行`moveToRetained()`任务,  从`watchedReferences`中移除这个元素,加入到另一个LinkedHashMap`retainedReferences`中. 然后调用`LeakSentryListener.onReferenceRetained()`

####思考: 这里为什么要延迟5秒执行任务
> 我们都知道GC不是即时的, 页面销毁后预留5秒的时间给GC操作, 再后续分析引用泄露, 避免无效的分析.

##4、LeakSentryListener.onReferenceRetained()
InternalLeakCanary实现了LeakSentryListener接口
```kotlin
internal object InternalLeakCanary : LeakSentryListener {
  
  override fun onReferenceRetained() {
    if (this::heapDumpTrigger.isInitialized) {
      heapDumpTrigger.onReferenceRetained()
    }
  }
}
```
HeapDumpTrigger#onReferenceRetained()里通过`backgroundHandler.post`把任务发送到HandlerThread中处理.
```kotlin
internal class HeapDumpTrigger{
  fun onReferenceRetained() {
    //上面提到的APP退到后台逻辑里,也会调用这个
    scheduleRetainedInstanceCheck("found new instance retained")
  }

  private fun scheduleRetainedInstanceCheck(reason: String) {
    //交给backgroundHandler发送到HandlerThread处理
    backgroundHandler.post {  
      checkRetainedInstances(reason)
    }
  }

  private fun checkRetainedInstances(reason: String) {
    CanaryLog.d("Checking retained instances because %s", reason)
    val config = configProvider()
    //如果LeakCanary.config的dumpHeap参数默认是true
    if (!config.dumpHeap) {
      return
    }
    //1.在GC之前, 筛选下retainedReferences中被回收掉的引用, 下面会说明
    var retainedKeys = refWatcher.retainedKeys
    //要检测的引用数量超过默认的数量, 就不处理了
    if (checkRetainedCount(retainedKeys, config.retainedVisibleThreshold)) return
    //如果程序在debug, 因为debug会保留之前的引用, 会对结果产生影响
    if (!config.dumpHeapWhenDebugging && DebuggerControl.isDebuggerAttached) {
      //这里等待20秒后再重试
      showRetainedCountWithDebuggerAttached(retainedKeys.size)
      scheduleRetainedInstanceCheck("debugger was attached", WAIT_FOR_DEBUG_MILLIS)
      CanaryLog.d(
          "Not checking for leaks while the debugger is attached, will retry in %d ms",
          WAIT_FOR_DEBUG_MILLIS
      )
      return
    }
    //2.手动触发GC
    gcTrigger.runGc()
    //3.在GC后, 再次筛选下retainedReferences中被回收掉的引用, 
    //到这一步剩下的就是没被回收掉的就是可能发生泄露的引用. 需要后续的dump分析
    retainedKeys = refWatcher.retainedKeys

    if (checkRetainedCount(retainedKeys, config.retainedVisibleThreshold)) return
    //4.保存这些没有被回收的对象的key, HeapAnalyzerService分析dump出的文件会用到
    HeapDumpMemoryStore.setRetainedKeysForHeapDump(retainedKeys)

    CanaryLog.d("Found %d retained references, dumping the heap", retainedKeys.size)
    HeapDumpMemoryStore.heapDumpUptimeMillis = SystemClock.uptimeMillis()
    dismissNotification()
    //5. 生成dump文件
    val heapDumpFile = heapDumper.dumpHeap()
    if (heapDumpFile == null) {
      //文件为空, 5秒后重试一次
      CanaryLog.d("Failed to dump heap, will retry in %d ms", WAIT_AFTER_DUMP_FAILED_MILLIS)
      scheduleRetainedInstanceCheck("failed to dump heap", WAIT_AFTER_DUMP_FAILED_MILLIS)
      showRetainedCountWithHeapDumpFailed(retainedKeys.size)
      return
    }
    //dump成功, 从retainedReferences.keys中删除这些key
    refWatcher.removeRetainedKeys(retainedKeys)
    //6. 开启Service分析dumpFile结果
    HeapAnalyzerService.runAnalysis(application, heapDumpFile)
  }
}
```
可以看到, HeapDumpTrigger#onReferenceRetained()先调用了一次
`refWatcher.retainedKeys`, 然后调用`gcTrigger.runGc()`进行GC, 接着又调用了一次`refWatcher.retainedKeys`
简单理一下.
1. 调用`refWatcher.retainedKeys` 得到残留的引用的key的集合`retainedKeys `
2. 调用gcTrigger.runGc()进行GC
3. 再次调用`refWatcher.retainedKeys`得到手动GC后残留的key的集合
4. 到这一步retainedKeys就是可能发生泄露的引用了,通过HeapDumpMemoryStore保存`retainedKeys`, 留着后面和dump结果比对
5.  通过`heapDumper.dumpHeap()`生成文件heapDumpFile
6. 开启服务分析dump结果和`retainedKeys`来判断是否真正泄露`HeapAnalyzerService.runAnalysis(application, heapDumpFile)`


#### 4.1 RefWatcher.retainedKeys
这一步是检查RefrenceQueue中是否出现了我们观察对象的弱引用, 如果出现了,说明被回收掉了, 那么就从 retainedReferences 删掉这个弱引用.
retainedReferences 最后剩下的是出现泄露需要dump的.
```kotlin
  val retainedKeys: Set<String>
    @Synchronized get() {
      // 删除弱可达的引用
      removeWeaklyReachableReferences()
      //返回retainedReferences 剩下的keys的队列
      return HashSet(retainedReferences.keys)
    }

  private fun removeWeaklyReachableReferences() {
    // WeakReferences are enqueued as soon as the object to which they point to becomes weakly
    // reachable. This is before finalization or garbage collection has actually happened.
    var ref: KeyedWeakReference?
    // 已经回收掉的弱引用会存放在RefrenceQueue中,循环移除
    do {
      ref = queue.poll() as KeyedWeakReference?
      //如果RefrenceQueue里存在,说明这个弱引用被回收了
      if (ref != null) {
        val removedRef = watchedReferences.remove(ref.key)
        //如果watchedReferences中的这个弱引用被回收了,retainedReferences也移除掉这个弱引用
        if (removedRef == null) {
          retainedReferences.remove(ref.key)
        }
      }
    } while (ref != null)
  }
```
这里是用**Refrence + RefrenceQueue**判断对象是否被回收的
ReferenceQueue : 当检测到对象的可达性更改后, 将会把对象加入ReferenceQueue引用队列.  
我们给每个被观测对象创建的`KeyedWeakReference(watchedReference, key, referenceName, watchUptimeMillis, queue)` 里, 就使用了**Refrence + RefrenceQueue**
```kotlin
class KeyedWeakReference(
  referent: Any,
  /**
   * Key used to find the retained references in the heap dump.
   */
  val key: String,
  val name: String,
  val watchUptimeMillis: Long,
  referenceQueue: ReferenceQueue<Any>
) : WeakReference<Any>(
    referent, referenceQueue
) {
  val className: String = referent.javaClass.name
}
```
> 当弱引用被回收后, 就会被加入我们构造他的时候关联的ReferenceQueue里,我们只要观测这个ReferenceQueue就能知道引用是否被回收了. LeakCanary就是这么干的. 更多原理可以参考[带你读懂 Reference 和 ReferenceQueue](https://blog.csdn.net/gdutxiaoxu/article/details/80738581)


#### 4.2 gcTrigger.runGc()
注释说‘system.gc()并不会每次都执行，参考AOSP中的的一段代码,使用Runtime.gc()更可能触发GC.
```kotlin
interface GcTrigger {

  fun runGc()

  object Default : GcTrigger {
    override fun runGc() {
      // Code taken from AOSP FinalizationTest:
      // https://android.googlesource.com/platform/libcore/+/master/support/src/test/java/libcore/
      // java/lang/ref/FinalizationTester.java
      // System.gc() does not garbage collect every time. Runtime.gc() is
      // more likely to perform a gc.
      Runtime.getRuntime()
          .gc()
      enqueueReferences()
      System.runFinalization()
    }

    private fun enqueueReferences() {
      // Hack. We don't have a programmatic way to wait for the reference queue daemon to move
      // references to the appropriate queues.
      try {
        Thread.sleep(100)
      } catch (e: InterruptedException) {
        throw AssertionError()
      }
    }
  }
}
```
#### 4.3 RefWatcher.retainedKeys再次调用
GC后, 继续遍历ReferenceQueue, 从retainedReferences中移除掉被回收掉的引用. 到这一步retainedReferences 剩下的就是可能发生泄露的引用了,需要dumpHeap来分析.

#### 4.4 HeapDumper#dumpHeap()
AndroidHeapDumper是 HeapDumper 的实现类, 调用`Debug.dumpHprofData(heapDumpFile.absolutePath)` 生成了hprof文件. 和我们手动用AS profiler生成的一样
```kotlin
internal class AndroidHeapDumper(
  context: Context,
  private val leakDirectoryProvider: LeakDirectoryProvider
) : HeapDumper {

  override fun dumpHeap(): File? {
    val heapDumpFile = leakDirectoryProvider.newHeapDumpFile() ?: return null

    val waitingForToast = FutureResult<Toast?>()
    showToast(waitingForToast)
    //这里用countDownLatch来实现, 感兴趣的可以自行查看
    if (!waitingForToast.wait(5, SECONDS)) {
      CanaryLog.d("Did not dump heap, too much time waiting for Toast.")
      return null
    }
    //通知栏提示正在dumpHeap
    val notificationManager =
      context.getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager
    if (Notifications.canShowNotification) {
      val dumpingHeap = context.getString(R.string.leak_canary_notification_dumping)
      val builder = Notification.Builder(context)
          .setContentTitle(dumpingHeap)
      val notification = Notifications.buildNotification(context, builder, LEAKCANARY_LOW)
      notificationManager.notify(R.id.leak_canary_notification_dumping_heap, notification)
    }

    val toast = waitingForToast.get()

    return try {
      //把 hprof 文件保存到 指定的path里
      Debug.dumpHprofData(heapDumpFile.absolutePath)
      if (heapDumpFile.length() == 0L) {
        CanaryLog.d("Dumped heap file is 0 byte length")
        null
      } else {
        heapDumpFile
      }
    } catch (e: Exception) {
      CanaryLog.d(e, "Could not dump heap")
      // Abort heap dump
      null
    } finally {
      cancelToast(toast)
      notificationManager.cancel(R.id.leak_canary_notification_dumping_heap)
    }
  }
}
```

## 5、HeapAnalyzerService.runAnalysis(application, heapDumpFile)
开启一个前台服务, 把上一步生成的heapDumpFile传进来分析

```kotlin
internal class HeapAnalyzerService : ForegroundService(

  fun runAnalysis(context: Context, heapDumpFile: File) {
      val intent = Intent(context, HeapAnalyzerService::class.java)
      intent.putExtra(HEAPDUMP_FILE_EXTRA, heapDumpFile)
      //这里使用前台服务配合Notification使用
      ContextCompat.startForegroundService(context, intent)
  }
}
```
HeapAnalyzerService是ForegroundService的子类, ForegroundService又是IntentSerice的子类,  这里的`onHandleIntentInForeground` 实际是在IntentService的`onHandleIntent()`中执行的
```kotlin
override fun onHandleIntent(intent: Intent?) {
  onHandleIntentInForeground(intent)
}
```
所以`onHandleIntentInForeground`是在子线程执行的.他主要做了两件事: 

1. `val heapAnalysis = heapAnalyzer.checkForLeaks(heapDumpFile...)` 分析上一步得到的heapDumpFile, 通过调用作者的另外一个库haha得到结果.
2. `config.analysisResultListener(application, heapAnalysis)` 展示分析的结果.就是我们使用LeakCanary看到的泄露引用链的页面

```kotlin
override fun onHandleIntentInForeground(intent: Intent?) {
    if (intent == null) {
      CanaryLog.d("HeapAnalyzerService received a null intent, ignoring.")
      return
    }
    // Since we're running in the main process we should be careful not to impact it.
    //因为2.0不再和以前一样开启一个新的进程来处理, 这里手动降低线程的优先级
    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND)
    val heapDumpFile = intent.getSerializableExtra(HEAPDUMP_FILE_EXTRA) as File
    val heapAnalyzer = HeapAnalyzer(this)
    val config = LeakCanary.config
    //根据dump文件分析内存泄露的结果
    val heapAnalysis = heapAnalyzer.checkForLeaks(
          heapDumpFile, config.exclusionsFactory, config.computeRetainedHeapSize,
          config.leakInspectors, config.labelers
      )

    try {
      //检测结束回调 通知栏展示
      config.analysisResultListener(application, heapAnalysis)
    } finally {
      //finally中删除dump文件
      heapAnalysis.heapDumpFile.delete()
    }
  }
```
## 6、HeapAnalyzer#checkForLeaks
根据dump后的hprof文件查找泄露的情况.

```kotlin
class HeapAnalyzer constructor(
  private val listener: AnalyzerProgressListener
) {

  /**
   * Searches the heap dump for a [KeyedWeakReference] instance with the corresponding key,
   * and then computes the shortest strong reference path from that instance to the GC roots.
   */
  fun checkForLeaks(
    heapDumpFile: File,
    exclusionsFactory: ExclusionsFactory = { emptyList() },
    computeRetainedHeapSize: Boolean = false,
    reachabilityInspectors: List<LeakInspector> = emptyList(),
    labelers: List<Labeler> = emptyList()
  ): HeapAnalysis {
    val analysisStartNanoTime = System.nanoTime()

    if (!heapDumpFile.exists()) {
      val exception = IllegalArgumentException("File does not exist: $heapDumpFile")
      return HeapAnalysisFailure(
          heapDumpFile, System.currentTimeMillis(), since(analysisStartNanoTime),
          HeapAnalysisException(exception)
      )
    }

    listener.onProgressUpdate(READING_HEAP_DUMP_FILE)

    try {
      // 使用haha库创建一个HprofParser, 他实现了Closeable接口可以用kotlin的use函数
      HprofParser.open(heapDumpFile)
          .use { parser ->
            //扫描中..
            listener.onProgressUpdate(SCANNING_HEAP_DUMP)
            
            val (gcRootIds, keyedWeakReferenceInstances, cleaners) = scan(
                parser, computeRetainedHeapSize
            )
            val analysisResults = mutableMapOf<String, RetainedInstance>()
            listener.onProgressUpdate(FINDING_WATCHED_REFERENCES)
            //从HeapDumpMemoryStore中读取dump之前存储的retainedKeys
            val (retainedKeys, heapDumpUptimeMillis) = readHeapDumpMemoryStore(parser)

            if (retainedKeys.isEmpty()) {
              val exception = IllegalStateException("No retained keys found in heap dump")
              return HeapAnalysisFailure(
                  heapDumpFile, System.currentTimeMillis(), since(analysisStartNanoTime),
                  HeapAnalysisException(exception)
              )
            }
            //找到泄露的引用
            val leakingWeakRefs =
              findLeakingReferences(
                  parser, retainedKeys, analysisResults, keyedWeakReferenceInstances,
                  heapDumpUptimeMillis
              )
            //找到泄露引用的最短路径
            val (pathResults, dominatedInstances) =
              findShortestPaths(
                  parser, exclusionsFactory, leakingWeakRefs, gcRootIds,
                  computeRetainedHeapSize
              )
            //计算泄露的大小
            val retainedSizes = if (computeRetainedHeapSize) {
              computeRetainedSizes(parser, pathResults, dominatedInstances, cleaners)
            } else {
              null
            }
            
            buildLeakTraces(
                reachabilityInspectors, labelers, pathResults, parser,
                leakingWeakRefs, analysisResults, retainedSizes
            )

            addRemainingInstancesWithNoPath(parser, leakingWeakRefs, analysisResults)

            return HeapAnalysisSuccess(
                heapDumpFile, System.currentTimeMillis(), since(analysisStartNanoTime),
                analysisResults.values.toList()
            )
          }
    } catch (exception: Throwable) {
      return HeapAnalysisFailure(
          heapDumpFile, System.currentTimeMillis(), since(analysisStartNanoTime),
          HeapAnalysisException(exception)
      )
    }
  }
}
```
## 7、LeakCanary.Config#analysisResultListener(application, heapAnalysis)
发送到notifycation,创建pendingIntent打开跳转到详细的泄露界面
```kotlin
object DefaultAnalysisResultListener : AnalysisResultListener {

   override fun invoke(application: Application,heapAnalysis: HeapAnalysis){
    val pendingIntent = LeakActivity.createPendingIntent(
        application, arrayListOf(GroupListScreen(), HeapAnalysisListScreen(), screenToShow)
    )

    val contentText = application.getString(R.string.leak_canary_notification_message)

    Notifications.showNotification(
        application, contentTitle, contentText, pendingIntent,
        R.id.leak_canary_notification_analysis_result,
        LEAKCANARY_RESULT
    )
  }
}
```
# 总结
* LeakCanary通过监听页面的声明周期, 在Activity或Fragment执行onDestroy()时候调用`refWatcher.watch()`
* 创建被观察对象的弱引用,加入`watchedReferences`, 预留5秒时间给GC再进行内存泄露的分析再检查内存的情况.
* 在HandlerThread中进行分析, 通过RefrenceQueue判断引用是否被回收.手动触发GC, 再判断引用是否被回收. 到这一步retainedReferences中剩下的就是可能会发生泄露的引用.
* 还是在HandlerThread中进行dump生成heapDumpFile, 开启一个前台服务去比对dump结果和retainedReferences.
* 使用haha库比对分析, 得到真正泄露的实例, 计算最短引用路径, 展示结果.