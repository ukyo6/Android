[EventBus](https://github.com/greenrobot/EventBus) 是一款在 Android 开发中使用的**发布/订阅事件**总线框架，基于观察者模式，将事件的接收者和发送者分开，简化了组件之间的通信，使用简单、效率高、体积小！下边是官方的 EventBus 原理图：



![img](https://user-gold-cdn.xitu.io/2018/4/27/1630655a5ebb73d4?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



EventBus 的用法可以参考[官网](http://greenrobot.org/eventbus/documentation/)，这里不做过多的说明。本文主要是从 EventBus 使用的方式入手，来分析 EventBus 背后的实现原理，以下内容基于**eventbus:3.1.1**版本，主要包括如下几个方面的内容：

- Subscribe注解
- 注册事件订阅方法
- 取消注册
- 发送事件
- 事件处理
- 粘性事件
- Subscriber Index
- 核心流程梳理

### 一、Subscribe注解

EventBus3.0 开始用`Subscribe`注解配置事件订阅方法，不再使用方法名了，例如：

```java
@Subscribe
public void handleEvent(String event) {
    // do something
}
```

其中事件类型可以是 Java 中已有的类型或者我们自定义的类型。 具体看下`Subscribe`注解的实现：

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)  //运行时注解
@Target({ElementType.METHOD})
public @interface Subscribe {
    // 指定事件订阅方法的线程模式，即在那个线程执行事件订阅方法处理事件，默认为POSTING
    ThreadMode threadMode() default ThreadMode.POSTING;
    // 是否支持粘性事件，默认为false
    boolean sticky() default false;
    // 指定事件订阅方法的优先级，默认为0，如果多个事件订阅方法可以接收相同事件的，则优先级高的先接收到事件
    int priority() default 0;
}
```

所以在使用`Subscribe`注解时可以根据需求指定`threadMode`、`sticky`、`priority`三个属性。

其中`threadMode`属性有如下几个可选值：

- **ThreadMode.POSTING**，默认的线程模式，在那个线程发送事件就在对应线程处理事件，避免了线程切换，效率高。
- **ThreadMode.MAIN**，如在主线程（UI线程）发送事件，则直接在主线程处理事件；如果在子线程发送事件，则先将事件入队列，然后通过 Handler 切换到主线程，依次处理事件。
- **ThreadMode.MAIN_ORDERED**，无论在那个线程发送事件，都先将事件入队列，然后通过 Handler 切换到主线程，依次处理事件。
- **ThreadMode.BACKGROUND**，如果在主线程发送事件，则先将事件入队列，然后通过线程池依次处理事件；如果在子线程发送事件，则直接在发送事件的线程处理事件。
- **ThreadMode.ASYNC**，无论在那个线程发送事件，都将事件入队列，然后通过线程池处理。

### 二、注册事件订阅方法

注册事件的方式如下：

```java
EventBus.getDefault().register(this);
```

其中`getDefault()`是一个[单例](https://www.jianshu.com/p/a8cdbfd9869e)方法，保证当前只有一个`EventBus`实例：

```java
public static EventBus getDefault() {
        if (defaultInstance == null) {
            synchronized (EventBus.class) {
                if (defaultInstance == null) {
                    defaultInstance = new EventBus();
                }
            }
        }
        return defaultInstance;
    }
```

继续看`new EventBus()`做了些什么：

```java
public EventBus() {
        this(DEFAULT_BUILDER);
    }
```

在这里又调用了`EventBus`的另一个构造函数来完成它相关属性的初始化：

```java
EventBus(EventBusBuilder builder) {
        logger = builder.getLogger();
        subscriptionsByEventType = new HashMap<>();
        typesBySubscriber = new HashMap<>();
        stickyEvents = new ConcurrentHashMap<>();
        mainThreadSupport = builder.getMainThreadSupport();
        mainThreadPoster = mainThreadSupport != null ? mainThreadSupport.createPoster(this) : null;
        backgroundPoster = new BackgroundPoster(this);
        asyncPoster = new AsyncPoster(this);
        indexCount = builder.subscriberInfoIndexes != null ? builder.subscriberInfoIndexes.size() : 0;
        subscriberMethodFinder = new SubscriberMethodFinder(builder.subscriberInfoIndexes,
                builder.strictMethodVerification, builder.ignoreGeneratedIndex);
        logSubscriberExceptions = builder.logSubscriberExceptions;
        logNoSubscriberMessages = builder.logNoSubscriberMessages;
        sendSubscriberExceptionEvent = builder.sendSubscriberExceptionEvent;
        sendNoSubscriberEvent = builder.sendNoSubscriberEvent;
        throwSubscriberException = builder.throwSubscriberException;
        eventInheritance = builder.eventInheritance;
        executorService = builder.executorService;
    }
```

`DEFAULT_BUILDER`就是一个默认的[EventBusBuilder](https://github.com/greenrobot/EventBus/blob/master/EventBus/src/org/greenrobot/eventbus/EventBusBuilder.java)：

```java
private static final EventBusBuilder DEFAULT_BUILDER = new EventBusBuilder();
```

如果有需要的话，我们也可以通过配置`EventBusBuilder`来更改`EventBus`的属性，例如用如下方式注册事件：

```java
EventBus.builder()
        .eventInheritance(false)
        .logSubscriberExceptions(false)
        .build()
        .register(this);
```

有了`EventBus`的实例就可以进行注册了：

```java
public void register(Object subscriber) {
        // 得到当前要注册类的Class对象
        Class<?> subscriberClass = subscriber.getClass();
        // 1.根据Class查找当前类中订阅了事件的方法集合，即使用了Subscribe注解、有public修饰符、一个参数的方法
        // SubscriberMethod类主要封装了符合条件方法的相关信息：
        // Method对象、线程模式、事件类型、优先级、是否是粘性事等
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            // 2.循环遍历订阅了事件的方法集合，以完成注册
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
    }
```

#### 1. 查找订阅方法 `findSubscriberMethods()`

可以看到`register()`方法主要分为查找和注册两部分，首先来看查找的过程，从`findSubscriberMethods()`开始：

```java
private static final Map<Class<?>, List<SubscriberMethod>> METHOD_CACHE = new ConcurrentHashMap<>();

List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        // METHOD_CACHE是一个ConcurrentHashMap，直接保存了subscriberClass和对应SubscriberMethod的集合，以提高注册效率，防止重复查找
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        if (subscriberMethods != null) {
            return subscriberMethods;
        }
        // 由于使用了默认的EventBusBuilder，则ignoreGeneratedIndex属性默认为false，不忽略注解生成器
        if (ignoreGeneratedIndex) {
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            subscriberMethods = findUsingInfo(subscriberClass);
        }
        // 如果对应类中没有符合条件的方法，则抛出异常
        if (subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
            // 保存查找到的订阅事件的方法
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
    }
```

`findSubscriberMethods()`流程很清晰，即先从缓存中查找，如果找到则直接返回，否则去做下一步的查找过程，然后缓存查找到的集合，根据上边的注释可知`findUsingInfo()`方法会被调用：

```java
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        // 初始状态下findState.clazz就是subscriberClass
        while (findState.clazz != null) {
            findState.subscriberInfo = getSubscriberInfo(findState);  //查找apt在编译期生成的信息
            // 如果有apt生成的订阅方法
            if (findState.subscriberInfo != null) {
                SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
                for (SubscriberMethod subscriberMethod : array) {
                    if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                        findState.subscriberMethods.add(subscriberMethod);
                    }
                }
            } else {
                // 没有apt生成的信息, 就反射查找订阅方法
                findUsingReflectionInSingleClass(findState);
            }
            // 修改findState.clazz为subscriberClass的父类Class，即需要遍历父类
            findState.moveToSuperclass();
        }
        // 查找到的方法保存在了FindState实例的subscriberMethods集合中。
        // 使用subscriberMethods构建一个新的List<SubscriberMethod>
        // 释放掉findState
        return getMethodsAndRelease(findState);
    }
```

`findUsingInfo()`方法会在当前要注册的类以及其父类中查找订阅事件的方法，这里出现了一个`FindState`类，它是`SubscriberMethodFinder`的内部类，用来辅助查找订阅事件的方法，具体的查找过程在`findUsingReflectionInSingleClass()`方法，它主要通过反射查找订阅事件的方法：

```java
private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        try {
            // This is faster than getMethods, especially when subscribers are fat classes like Activities
            // getDeclaredMethods: 类所有自己的方法, 不包含继承的方法, 查找效率比 getMethods() 高
            // getMethods: 类和父类的所有公共方法
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
            methods = findState.clazz.getMethods();
            findState.skipSuperClasses = true;
        }
        // 循环遍历当前类的方法，筛选出符合条件的
        for (Method method : methods) {
            // 获得方法的修饰符
            int modifiers = method.getModifiers();
            // 如果是public类型，但非abstract、static等
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                // 获得当前方法所有参数的类型
                Class<?>[] parameterTypes = method.getParameterTypes();
                // 如果当前方法只有一个参数
                if (parameterTypes.length == 1) {
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    // 如果当前方法使用了Subscribe注解
                    if (subscribeAnnotation != null) {
                        // 得到该参数的类型
                        Class<?> eventType = parameterTypes[0];
                        // checkAdd()方法用来判断FindState的anyMethodByEventType map是否已经添加过以当前eventType为key的键值对，没添加过则返回true
                        if (findState.checkAdd(method, eventType)) {
                             // 得到Subscribe注解的threadMode属性值，即线程模式
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
                            // 创建一个SubscriberMethod对象，并添加到subscriberMethods集合
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                    subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                        }
                    }
                } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                    String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                    throw new EventBusException("@Subscribe method " + methodName +
                            "must have exactly 1 parameter but has " + parameterTypes.length);
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException(methodName +
                        " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
            }
        }
    }
```

到此`register()`方法中`findSubscriberMethods()`流程就分析完了，我们已经找到了当前注册类及其父类中订阅事件的方法的集合。接下来分析具体的注册流程，即`register()`中的`subscribe()`方法：

#### 2. 注册订阅 `subscribe(subscriberClass, subscriberMethods)`

```java
 private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
        // 得到当前订阅了事件的方法的参数类型
        Class<?> eventType = subscriberMethod.eventType;
        // Subscription类保存了要注册的类对象以及当前的subscriberMethod
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
        // subscriptionsByEventType是hashMap
        // key = eventType, value = CopyOnWriteArrayList<Subscription>>
        CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        // 如果不存在，则创建一个subscriptions，并保存到subscriptionsByEventType
        if (subscriptions == null) {
            subscriptions = new CopyOnWriteArrayList<>();
            subscriptionsByEventType.put(eventType, subscriptions);
        } else {
            if (subscriptions.contains(newSubscription)) {  
                throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                        + eventType);
            }
        }
        // 添加上边创建的newSubscription对象到subscriptions集合中
        int size = subscriptions.size();
        for (int i = 0; i <= size; i++) {
            //插入的时候, 根据注解里设置的prority排序!!!!!!!!!!!!
            if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                subscriptions.add(i, newSubscription);
                break;
            }
        }
        // typesBySubscribere也是一个HashMap，保存了以当前要注册类的对象为key，注册类中订阅事件的方法的参数类型的集合为value的键值对
        // 查找是否存在对应的参数类型集合
        List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
        // 不存在则创建一个subscribedEvents，并保存到typesBySubscriber
        if (subscribedEvents == null) {
            subscribedEvents = new ArrayList<>();
            typesBySubscriber.put(subscriber, subscribedEvents);
        }
        // 保存当前订阅了事件的方法的参数类型
        subscribedEvents.add(eventType);
        // 粘性事件相关的，后边具体分析
        if (subscriberMethod.sticky) {
            if (eventInheritance) {
                // Existing sticky events of all subclasses of eventType have to be considered.
                // Note: Iterating over all events may be inefficient with lots of sticky events,
                // thus data structure should be changed to allow a more efficient lookup
                // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
                Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                for (Map.Entry<Class<?>, Object> entry : entries) {
                    Class<?> candidateEventType = entry.getKey();
                    if (eventType.isAssignableFrom(candidateEventType)) {
                        Object stickyEvent = entry.getValue();
                        checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                    }
                }
            } else {
                Object stickyEvent = stickyEvents.get(eventType);
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }
    }
```

这就是注册的核心流程，所以`subscribe()`方法主要是得到了`subscriptionsByEventType`、`typesBySubscriber`两个 HashMap。我们在发送事件的时候要用到`subscriptionsByEventType`，完成事件的处理。当取消 EventBus 注册的时候要用到`typesBySubscriber`、`subscriptionsByEventType`，完成相关资源的释放。

### 三、取消注册

接下来看，EventBus 如何取消注册：

```java
EventBus.getDefault().unregister(this);
```

核心的方法就是`unregister()`：

```java
public synchronized void unregister(Object subscriber) {
        // 得到当前注册类对象 对应的 订阅事件方法的参数类型 的集合
        List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
        if (subscribedTypes != null) {
            // 遍历参数类型集合，释放之前缓存的当前类中的Subscription
            for (Class<?> eventType : subscribedTypes) {
                unsubscribeByEventType(subscriber, eventType);
            }
            // 删除以subscriber为key的键值对
            typesBySubscriber.remove(subscriber);
        } else {
            logger.log(Level.WARNING, "Subscriber to unregister was not registered before: " + subscriber.getClass());
        }
    }
```

内容很简单，继续看`unsubscribeByEventType()`方法：

```java
private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
        // 得到当前参数类型对应的Subscription集合
        List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        if (subscriptions != null) {
            int size = subscriptions.size();
            // 遍历Subscription集合
            for (int i = 0; i < size; i++) {
                Subscription subscription = subscriptions.get(i);
                // 如果当前subscription对象对应的注册类对象 和 要取消注册的注册类对象相同，则删除当前subscription对象
                if (subscription.subscriber == subscriber) {
                    subscription.active = false;
                    subscriptions.remove(i);
                    i--;
                    size--;
                }
            }
        }
    }
```

所以在`unregister()`方法中，释放了`typesBySubscriber`、`subscriptionsByEventType`中缓存的资源。

### 四、发送事件

当发送一个事件的时候，我们可以通过如下方式：

```java
EventBus.getDefault().post("Hello World!")
```

可以看到，发送事件就是通过`post()`方法完成的：

```java
public void post(Object event) {
        // currentPostingThreadState是一个PostingThreadState类型的ThreadLocal
        // PostingThreadState类保存了事件队列和线程模式等信息
        PostingThreadState postingState = currentPostingThreadState.get();
        List<Object> eventQueue = postingState.eventQueue;
        // 将要发送的事件添加到事件队列
        eventQueue.add(event);
        // isPosting默认为false
        if (!postingState.isPosting) {
            // 是否为主线程
            postingState.isMainThread = isMainThread();
            postingState.isPosting = true;
            if (postingState.canceled) {
                throw new EventBusException("Internal error. Abort state was not reset");
            }
            try {
                // 遍历事件队列
                while (!eventQueue.isEmpty()) {
                    // 发送单个事件
                    // eventQueue.remove(0)，从事件队列移除事件
                    postSingleEvent(eventQueue.remove(0), postingState);
                }
            } finally {
                postingState.isPosting = false;
                postingState.isMainThread = false;
            }
        }
    }
```

所以`post()`方法先将发送的事件保存的事件队列，然后通过循环出队列，将事件交给`postSingleEvent()`方法处理：

```java
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
        Class<?> eventClass = event.getClass();
        boolean subscriptionFound = false;
        // eventInheritance默认为true，表示是否向上查找事件的父类
        if (eventInheritance) {
            // 查找当前类型, 和他的超类,接口的所有class
            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
            int countTypes = eventTypes.size();
            // 遍历Class集合，继续处理事件
            for (int h = 0; h < countTypes; h++) {
                Class<?> clazz = eventTypes.get(h);
                subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
            }
        } else {
            subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
        }
        if (!subscriptionFound) {
            if (logNoSubscriberMessages) {
                logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
            }
            if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                    eventClass != SubscriberExceptionEvent.class) {
                post(new NoSubscriberEvent(this, event));
            }
        }
    }
```

`postSingleEvent()`方法中，根据`eventInheritance`属性，决定是否向上遍历事件的父类型，然后用`postSingleEventForEventType()`方法进一步处理事件：

```java
private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
        CopyOnWriteArrayList<Subscription> subscriptions;
        synchronized (this) {
            // 获取事件类型对应的Subscription集合
            subscriptions = subscriptionsByEventType.get(eventClass);
        }
        // 如果已订阅了对应类型的事件
        if (subscriptions != null && !subscriptions.isEmpty()) {
            for (Subscription subscription : subscriptions) {
                // 记录事件
                postingState.event = event;
                // 记录对应的subscription
                postingState.subscription = subscription;
                boolean aborted = false;
                try {
                    // 最终的事件处理
                    postToSubscription(subscription, event, postingState.isMainThread);
                    aborted = postingState.canceled;
                } finally {
                    postingState.event = null;
                    postingState.subscription = null;
                    postingState.canceled = false;
                }
                if (aborted) {
                    break;
                }
            }
            return true;
        }
        return false;
    }
```

`postSingleEventForEventType()`方法核心就是遍历发送的事件类型对应的Subscription集合，然后调用`postToSubscription()`方法处理事件。

### 五、处理事件

接着上边的继续分析，`postToSubscription()`内部会根据订阅事件方法的线程模式，间接或直接的以发送的事件为参数，通过反射执行订阅事件的方法。

```java
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
        // 判断订阅事件方法的线程模式
        switch (subscription.subscriberMethod.threadMode) {
            // 默认的线程模式，在那个线程发送事件就在那个线程处理事件
            case POSTING:
                invokeSubscriber(subscription, event);
                break;
            // 在主线程处理事件
            case MAIN:
                // 如果在主线程发送事件，则直接在主线程通过反射处理事件
                if (isMainThread) {
                    invokeSubscriber(subscription, event);
                } else {
                    // 如果是在子线程发送事件，则将事件入队列，通过Handler切换到主线程执行处理事件
                    // mainThreadPoster 不为空
                    mainThreadPoster.enqueue(subscription, event);
                }
                break;
            // 无论在那个线程发送事件，都先将事件入队列，然后通过 Handler 切换到主线程，依次处理事件。
            // mainThreadPoster 不为空
            case MAIN_ORDERED:
                if (mainThreadPoster != null) {
                    mainThreadPoster.enqueue(subscription, event);
                } else {
                    invokeSubscriber(subscription, event);
                }
                break;
            case BACKGROUND:
                // 如果在主线程发送事件，则先将事件入队列，然后通过线程池依次处理事件
                if (isMainThread) {
                    backgroundPoster.enqueue(subscription, event);
                } else {
                    // 如果在子线程发送事件，则直接在发送事件的线程通过反射处理事件
                    invokeSubscriber(subscription, event);
                }
                break;
            // 无论在那个线程发送事件，都将事件入队列，然后通过线程池处理。
            case ASYNC:
                asyncPoster.enqueue(subscription, event);
                break;
            default:
                throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
        }
    }
```

可以看到，`postToSubscription()`方法就是根据订阅事件方法的线程模式、以及发送事件的线程来判断如何处理事件，至于处理方式主要有两种： 一种是在相应线程直接通过`invokeSubscriber()`方法，用反射来执行订阅事件的方法，这样发送出去的事件就被订阅者接收并做相应处理了：

```java
void invokeSubscriber(Subscription subscription, Object event) {
        try {
            subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
        } catch (InvocationTargetException e) {
            handleSubscriberException(subscription, event, e.getCause());
        } catch (IllegalAccessException e) {
            throw new IllegalStateException("Unexpected exception", e);
        }
    }
```

另外一种是先将事件入队列（其实底层是一个List），然后做进一步处理，我们以`mainThreadPoster.enqueue(subscription, event)`为例简单的分析下，其中`mainThreadPoster`是`HandlerPoster`类的一个实例，来看该类的主要实现：

```java
public class HandlerPoster extends Handler implements Poster {
    private final PendingPostQueue queue;
    private boolean handlerActive;
    ......
    public void enqueue(Subscription subscription, Object event) {
        // 用subscription和event封装一个PendingPost对象
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        synchronized (this) {
            // 入队列
            queue.enqueue(pendingPost);
            if (!handlerActive) {
                handlerActive = true;
                // 发送开始处理事件的消息，handleMessage()方法将被执行，完成从子线程到主线程的切换
                if (!sendMessage(obtainMessage())) {
                    throw new EventBusException("Could not send handler message");
                }
            }
        }
    }

    @Override
    public void handleMessage(Message msg) {
        boolean rescheduled = false;
        try {
            long started = SystemClock.uptimeMillis();
            // 死循环遍历队列
            while (true) {
                // 出队列
                PendingPost pendingPost = queue.poll();
                ......
                // 进一步处理pendingPost
                eventBus.invokeSubscriber(pendingPost);
                ......
            }
        } finally {
            handlerActive = rescheduled;
        }
    }
}
```

所以`HandlerPoster`的`enqueue()`方法主要就是将`subscription`、`event`对象封装成一个`PendingPost`对象，然后保存到队列里，之后通过`Handler`切换到主线程，在`handleMessage()`方法将中将`PendingPost`对象循环出队列，交给`invokeSubscriber()`方法进一步处理：

```java
void invokeSubscriber(PendingPost pendingPost) {
        Object event = pendingPost.event;
        Subscription subscription = pendingPost.subscription;
        // 释放pendingPost引用的资源
        PendingPost.releasePendingPost(pendingPost);
        if (subscription.active) {
            // 用反射来执行订阅事件的方法
            invokeSubscriber(subscription, event);
        }
    }
```

这个方法很简单，主要就是从`pendingPost`中取出之前保存的`event`、`subscription`，然后用反射来执行订阅事件的方法，又回到了第一种处理方式。所以`mainThreadPoster.enqueue(subscription, event)`的核心就是先将将事件入队列，然后通过Handler从子线程切换到主线程中去处理事件。

`backgroundPoster.enqueue()`和`asyncPoster.enqueue`也类似，内部都是先将事件入队列，然后再出队列，但是会通过线程池去进一步处理事件。

### 六、粘性事件

一般情况，我们使用 EventBus 都是准备好订阅事件的方法，然后注册事件，最后在发送事件，即要先有事件的接收者。但粘性事件却恰恰相反，我们可以先发送事件，后续再准备订阅事件的方法、注册事件。

由于这种差异，我们分析粘性事件原理时，先从事件发送开始，发送一个粘性事件通过如下方式：

```java
EventBus.getDefault().postSticky("Hello World!");
```

来看`postSticky()`方法是如何实现的：

```java
public void postSticky(Object event) {
        synchronized (stickyEvents) {
            stickyEvents.put(event.getClass(), event);
        }
        post(event);
    }
```

`postSticky()`方法主要做了两件事，先将事件类型和对应事件保存到`stickyEvents`中，方便后续使用；然后执行`post(event)`继续发送事件，这个`post()`方法就是之前发送的`post()`方法。所以，如果在发送粘性事件前，已经有了对应类型事件的订阅者，及时它是非粘性的，依然可以接收到发送出的粘性事件。

发送完粘性事件后，再准备订阅粘性事件的方法，并完成注册。核心的注册事件流程还是我们之前的`register()`方法中的`subscribe()`方法，前边分析`subscribe()`方法时，有一段没有分析的代码，就是用来处理粘性事件的：

```java
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
        ......
        ......
        ......
        // 如果当前订阅事件的方法的Subscribe注解的sticky属性为true，即该方法可接受粘性事件
        if (subscriberMethod.sticky) {
            // 默认为true，表示是否向上查找事件的父类
            if (eventInheritance) {
                // stickyEvents就是发送粘性事件时，保存了事件类型和对应事件
                Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                for (Map.Entry<Class<?>, Object> entry : entries) {
                    Class<?> candidateEventType = entry.getKey();
                    // 如果candidateEventType是eventType的子类或
                    if (eventType.isAssignableFrom(candidateEventType)) {
                        // 获得对应的事件
                        Object stickyEvent = entry.getValue();
                        // 处理粘性事件
                        checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                    }
                }
            } else {
                Object stickyEvent = stickyEvents.get(eventType);
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }
    }
```

可以看到，处理粘性事件就是在 EventBus 注册时，遍历`stickyEvents`，如果当前要注册的事件订阅方法是粘性的，并且该方法接收的事件类型和`stickyEvents`中某个事件类型相同或者是其父类，则取出`stickyEvents`中对应事件类型的具体事件，做进一步处理。继续看`checkPostStickyEventToSubscription()`处理方法：

```java
private void checkPostStickyEventToSubscription(Subscription newSubscription, Object stickyEvent) {
        if (stickyEvent != null) {
            postToSubscription(newSubscription, stickyEvent, isMainThread());
        }
    }
```

最终还是通过`postToSubscription()`方法完成粘性事件的处理，这就是粘性事件的整个处理流程。

### 七、Subscriber Index

回顾之前分析的 EventBus 注册事件流程，主要是在项目运行时通过反射来查找订事件的方法信息，这也是默认的实现，如果项目中有大量的订阅事件的方法，必然会对项目运行时的性能产生影响。其实除了在项目运行时通过反射查找订阅事件的方法信息，EventBus 还提供了在项目编译时通过[注解处理器](https://github.com/greenrobot/EventBus/blob/master/EventBusAnnotationProcessor/src/org/greenrobot/eventbus/annotationprocessor/EventBusAnnotationProcessor.java)查找订阅事件方法信息的方式，生成一个辅助的索引类来保存这些信息，这个索引类就是[Subscriber Index](http://greenrobot.org/eventbus/documentation/subscriber-index/)，其实和 [ButterKnife]() 的原理类似。

要在项目编译时查找订阅事件的方法信息，首先要在 app 的 build.gradle 中加入如下配置：

```groovy
android {
    defaultConfig {
        javaCompileOptions {
            annotationProcessorOptions {
                // 根据项目实际情况，指定辅助索引类的名称和包名
                arguments = [ eventBusIndex : 'com.shh.sometest.MyEventBusIndex' ]
            }
        }
    }
}
dependencies {
    compile 'org.greenrobot:eventbus:3.1.1'
    // 引入注解处理器
    annotationProcessor 'org.greenrobot:eventbus-annotation-processor:3.1.1'
}
```

然后在项目的 Application 中添加如下配置，以生成一个默认的 EventBus 单例：

```
EventBus.builder().addIndex(new MyEventBusIndex()).installDefaultEventBus();
```

之后的用法就和我们平时使用 EventBus 一样了。当项目编译时会在生成对应的`MyEventBusIndex`类： 

![MyEventBusIndex](https://user-gold-cdn.xitu.io/2018/4/27/1630655a5ed20b8a?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

 对应的源码如下：

```java
public class MyEventBusIndex implements SubscriberInfoIndex {
    private static final Map<Class<?>, SubscriberInfo> SUBSCRIBER_INDEX;

    static {
        SUBSCRIBER_INDEX = new HashMap<Class<?>, SubscriberInfo>();

        putIndex(new SimpleSubscriberInfo(MainActivity.class, true, new SubscriberMethodInfo[] {
            new SubscriberMethodInfo("changeText", String.class),
        }));

    }

    private static void putIndex(SubscriberInfo info) {
        SUBSCRIBER_INDEX.put(info.getSubscriberClass(), info);
    }

    @Override
    public SubscriberInfo getSubscriberInfo(Class<?> subscriberClass) {
        SubscriberInfo info = SUBSCRIBER_INDEX.get(subscriberClass);
        if (info != null) {
            return info;
        } else {
            return null;
        }
    }
}
```

其中`SUBSCRIBER_INDEX`是一个HashMap，保存了当前注册类的 Class 类型和其中事件订阅方法的信息。

接下来分析下使用 Subscriber Index 时 EventBus 的注册流程，我们先分析：

```java
EventBus.builder().addIndex(new MyEventBusIndex()).installDefaultEventBus();
```

首先创建一个`EventBusBuilder`，然后通过`addIndex()`方法添加索引类的实例：

```java
public EventBusBuilder addIndex(SubscriberInfoIndex index) {
        if (subscriberInfoIndexes == null) {
            subscriberInfoIndexes = new ArrayList<>();
        }
        subscriberInfoIndexes.add(index);
        return this;
    }
```

即把生成的索引类的实例保存在`subscriberInfoIndexes`集合中，然后用`installDefaultEventBus()`创建默认的 EventBus实例：

```java
public EventBus installDefaultEventBus() {
        synchronized (EventBus.class) {
            if (EventBus.defaultInstance != null) {
                throw new EventBusException("Default instance already exists." +
                        " It may be only set once before it's used the first time to ensure consistent behavior.");
            }
            EventBus.defaultInstance = build();
            return EventBus.defaultInstance;
        }
    }

public EventBus build() {
        // this 代表当前EventBusBuilder对象
        return new EventBus(this);
    }
```

即用当前`EventBusBuilder`对象创建一个 EventBus 实例，这样我们通过`EventBusBuilder`配置的 Subscriber Index 也就传递到了EventBus实例中，然后赋值给EventBus的 `defaultInstance`成员变量。之前我们在分析 EventBus 的`getDefault()`方法时已经见到了`defaultInstance`：

```java
public static EventBus getDefault() {
        if (defaultInstance == null) {
            synchronized (EventBus.class) {
                if (defaultInstance == null) {
                    defaultInstance = new EventBus();
                }
            }
        }
        return defaultInstance;
    }
```

所以在 Application 中生成了 EventBus 的默认单例，这样就保证了在项目其它地方执行`EventBus.getDefault()`就能得到唯一的 EventBus 实例！之前在分析注册流程时有一个 方法`findUsingInfo()`：

```java
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            // 查找SubscriberInfo
            findState.subscriberInfo = getSubscriberInfo(findState);
            // 条件成立
            if (findState.subscriberInfo != null) {
                // 获得当前注册类中所有订阅了事件的方法
                SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
                for (SubscriberMethod subscriberMethod : array) {
                    // findState.checkAdd()之前已经分析过了，即是否在FindState的anyMethodByEventType已经添加过以当前eventType为key的键值对，没添加过返回true
                    if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                        // 将subscriberMethod对象添加到subscriberMethods集合
                        findState.subscriberMethods.add(subscriberMethod);
                    }
                }
            } else {
                findUsingReflectionInSingleClass(findState);
            }
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }
```

由于我们现在使用了 Subscriber Index 所以不会通过`findUsingReflectionInSingleClass()`来反射解析订阅事件的方法。我们重点来看`getSubscriberInfo()`都做了些什么：

```java
private SubscriberInfo getSubscriberInfo(FindState findState) {
        // 该条件不成立
        if (findState.subscriberInfo != null && findState.subscriberInfo.getSuperSubscriberInfo() != null) {
            SubscriberInfo superclassInfo = findState.subscriberInfo.getSuperSubscriberInfo();
            if (findState.clazz == superclassInfo.getSubscriberClass()) {
                return superclassInfo;
            }
        }
        // 该条件成立
        if (subscriberInfoIndexes != null) {
            // 遍历索引类实例集合
            for (SubscriberInfoIndex index : subscriberInfoIndexes) {
                // 根据注册类的 Class 类查找SubscriberInfo
                SubscriberInfo info = index.getSubscriberInfo(findState.clazz);
                if (info != null) {
                    return info;
                }
            }
        }
        return null;
    }
```

`subscriberInfoIndexes`就是在前边`addIndex()`方法中创建的，保存了项目中的索引类实例，即`MyEventBusIndex`的实例，继续看索引类的`getSubscriberInfo()`方法，来到了`MyEventBusIndex`类中：

```java
@Override
    public SubscriberInfo getSubscriberInfo(Class<?> subscriberClass) {
        SubscriberInfo info = SUBSCRIBER_INDEX.get(subscriberClass);
        if (info != null) {
            return info;
        } else {
            return null;
        }
    }
```

即根据注册类的 Class 类型从 `SUBSCRIBER_INDEX` 查找对应的`SubscriberInfo`，如果我们在注册类中定义了订阅事件的方法，则 `info`不为空，进而上边`findUsingInfo()`方法中`findState.subscriberInfo != null`成立，到这里主要的内容就分析完了，其它的和之前的注册流程一样。

所以 Subscriber Index 的核心就是项目编译时使用注解处理器生成保存事件订阅方法信息的索引类，然后项目运行时将索引类实例设置到 EventBus 中，这样当注册 EventBus 时，从索引类取出当前注册类对应的事件订阅方法信息，以完成最终的注册，避免了运行时反射处理的过程，所以在性能上会有质的提高。项目中可以根据实际的需求决定是否使用 Subscriber Index。

### 八、小结

结合上边的分析，我们可以总结出`register`、`post`、`unregister`的核心流程：



![register](https://user-gold-cdn.xitu.io/2018/4/27/1630655a5f017123?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)





![post](https://user-gold-cdn.xitu.io/2018/4/27/1630655a5f12b2d7?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)





![unregister](https://user-gold-cdn.xitu.io/2018/4/27/1630655a5f7e683b?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



到这里 EventBus 几个重要的流程就分析完了，整体的设计思路还是值得我们学习的。和 Android 自带的广播相比，使用简单、同时又兼顾了性能。但如果在项目中滥用的话，各种逻辑交叉在一起，也可能会给后期的维护带来困难。


作者：SheHuan链接：https://juejin.im/post/5ae2e6dcf265da0b9d77f28e来源：掘金著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。