H5页面承载了懂球帝文章、活动、广告等核心业务场景，所以经过了长期的迭代之后，懂球帝客户端H5相关的业务也非常复杂，这里面包含了分享、支付、用户评论、点赞等交互，各种业务交织杂糅在一起，导致这一块的代码难以维护。笔者对业务进行了全面的梳理，在重构这块业务的过程中也收获了很多，同时考虑到很多产品都有相似的应用场景，分享出来希望对大家有帮助。 介绍下全文的结构和思路，首先笔者对现有业务进行抽象，提取出其中的调用关系和信息流，第二部分简单地介绍了项目中之前采用的方案和缺点。第三部分着重介绍了新的设计架构。第四部分则是基于新的架构，实现了一些优化的功能，比如缓存和HttpDns的功能。

#### 一、业务场景

首先我们分析一下一般App中，H5相关的业务场景和这些业务场景中的信息流。H5业务场景里面包含了几个角色：**WebView**是显示H5的控件；Activity/Fragment等嵌入有WebView的**界面**、**H5页面**、**JsBridge**是H5和客户端互相调用的桥梁，这里面用户直接打交道的是界面和H5页面。基本上，大部分业务场景可以归纳为以下几种

1. 用户与H5交互，需要调用客户端某项功能，比如用户点击了H5的按钮参加某个活动，这时候H5需要获取客户端用户的登录信息。



![img](https://user-gold-cdn.xitu.io/2019/12/7/16ee02e5c0d8ba6c?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



1. 用户与界面交互，需要调用H5的某项功能，比如用户下拉刷新，这时候需要通知H5去对数据做更新



![img](https://user-gold-cdn.xitu.io/2019/12/7/16ee02e85424cc74?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



1. 界面的某些非用户行为触发H5的相关操作，比如在Acitivty的生命周期中调用WebView或者H5页面进行一些操作



![img](https://user-gold-cdn.xitu.io/2019/12/7/16ee02ec10215ee3?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



1. Webview自身的状态改变需要调用界面，比如在WebView中WebviewClient和WebChromeClient的几个回调`onPageStarted、onPageFinished`等方法，需要分别通知界面和H5进行处理



![img](https://user-gold-cdn.xitu.io/2019/12/7/16ee02edfe76c583?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



业务逻辑可能是以上四种业务场景之一，复杂的业务场景也可能是这几种业务场景的组合，比如用户点击H5中的点赞按钮，这时候调用界面的登录功能，登录完成后，界面调用H5的功能执行刷新界面等操作。

#### 二、原有实现逻辑

原有的实现逻辑也比较简单粗暴，就是在界面和Webview中间增加一个WebviewManager，负责这两者之间的通信。



![img](https://user-gold-cdn.xitu.io/2019/12/7/16ee02ef9f2640d6?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



在WebView和H5之间增加一个BrigeHelper负责管理JsBridge的交互。一开始可能没什么大问题，但是随着业务代码的积累，就会发现几个问题：

1. WebviewManager和BrigeHelper的功能越来越多，越来越多的处理逻辑和代码往这里面堆
2. 由于通信是双向的，也就意味着WebviewManager和Bridgehelper必须提供双向通信的接口，但是随着业务的增长，接口也臃肿不堪，接口粒度过大导致有些页面只需要接口中的部分信息，也被迫去实现整个接口。而且，一有新的功能就必须要改接口，这也不符合设计规范
3. 所有业务都糅合在WebviewManager和BrigeHelper中，导致各个业务不好拆分甚至会互相影响。比如登录和支付之后返回当前页面都可能会有刷新页面的需求，这两个业务刷新页面的部分就杂糅在一起，天长日久后，就没人知道这一块业务到底是做什么的
4. WebviewManager和BridgeHelper没有任何约束，中间各个对象互相调用，有着千丝万缕的联系
5. 各个页面对Webview的使用方式也不太一致

#### 三、重构的方案

通过前面的业务介绍可以发现，H5和native之间的交互多且杂。笔者在项目中也尝试去使用[Cordova](https://github.com/apache/cordova-android)的方案，但是Cordova过于庞大复杂，客户端和前端要完全迁移到这上面，特别是对于懂球帝这样迭代了几年的项目，成本过高。出于实际的考虑，笔者准备探寻其他的解决方案。不过看了Cordova的源码也给了我一定的思路。 通过进一步对业务场景的抽象，笔者初步倾向于把很多业务以及JsBridge都抽离出来互相独立，比如分享的业务、支付的业务、用户相关的业务等，这种情境就非常适合用策略模式，使用策略模式有几个优点：

- 把各个业务封装在独立的策略中就可以不互相影响；
- 策略也可以自由组合，给界面和H5提供不同的功能；
- 策略模式可以方便扩展，扩展策略接口就好了

当然策略模式也有缺点，就是使用者必须要了解各个业务，而且策略模式会多出来很多类，目前看，应该是优点大于缺点



![img](https://user-gold-cdn.xitu.io/2019/12/7/16ee02f4a41708d6?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

 定下来使用策略模式后，就面临一个问题，如何定义策略接口。简单的策略模式接口只有一个接口，但是在我们的场景中，策略的触发会比较复杂，在第一节中我们就分析发现，策略可能是用户触发的、界面的生命周期触发、H5通过JsBridge的调用以及WebViewClient等相关回调触发，所以我们要对策略模式做一些变形，将一个接口拆分成多个接口,然后将策略的接口改为抽象类实现：

- Activity/Fragment的生命周期可以从界面的LifeCycle中获得，实现 LifeCycleObserver 就可以了
- Webview的触发事件定义为IWebviewCallback，这里面包含策略里面需要用到的主要方法

```java
public interface IWebviewCallback {
    void onPageFinished();
    void onPageStart();
    boolean shouldOverrideUrlLoading(WebView webView, String url);
    WebResourceResponse shouldInterceptRequest(WebView view, final WebResourceRequest webResourceRequest);
    boolean onShowFileChooser(ValueCallback<Uri[]> filePathCallback,WebChromeClient.FileChooserParamsfileChooserParams);
    void onLoadResource(WebView webView, String url);
    ......
}
```

- JsBridge触发的定义为IBridgeHandler

```java
public interface IBridgeHandler {
    /**
     * bridge名称
     * @return
     */
    public String[] getName();

    /**
     * Birdge触发时调用调用
     * @param jsonObject
     * @param callBackFunction
     */
    public void onHandle(String functionName, JSONObject jsonObject, CallBackFunction callBackFunction);
}
```

- 策略自身被加载和卸载时的生命周期的监听

```java
public interface IPluginLifeCycle {
    /**
     * 插件被添加的时候调用
     */
    void onPluginAdded();
    /**
     * 插件被移除的时候调用
     */
    void onPluginRemoved();
}
```

接下来定义策略的接口IWebviewplugin继承者四个接口

```java
public abstract class IWebviewPlugin implements GenericLifecycleObserver, IPluginLifeCycle, IWebviewCallback, IBridgeHandler {
}
```

这样策略的接口就定义好了，并且具备了处理来自WebView、界面生命周期、JsBridge以及自身生命周期的能力。通过这四个接口，我们就能实现第一部分中介绍的业务场景的功能 接口定义好之后，整个框架就呼之欲出了：

![img](https://user-gold-cdn.xitu.io/2019/12/7/16ee02f77318f0f9?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



除了以上介绍的几个接口，我们还需要几个接口和类：

- `WebHostCallback`：由界面实现，可以获取WebView的状态
- `PageInterface`: 由于使用WebView的可能是Activity，也有可能是Fragment甚至是一个ViewGroup，所以需要对界面的功能进行抽象，提供一个可供WebView使用的一些功能，比如打开一个Activity等
- `PluginManager`：策略的管理类，可以动态组合管理策略集合
- `WebviewWrapper`：WebView的包装类，这里采用抽象WebView的一些Api然后包装的形式而不是继承WebView来扩展功能，主要是考虑到使用者可能继承WebView实现自己的一些功能，甚至是未来可能更换系统的WebView采用第三方的内核
- `PluginFactory`：生产策略的工厂

可以看到，使用接口抽象后，WebView并不知道有多少个策略被实现了，它要做的只是调用IWebviewCallback接口，这样就把WebView从业务中抽离出来，WebviewWrapper并不涉及到业务的代码，只需要配置和管理好WebView就可以了 通过PageInterface和WebviewHostCallback的接口，Webview也不需要关心自己是在Activity还是Fragment里面。 从使用者的角度来看，继承IWebviewPlugin实现自己的策略就可以了。 通过接口抽象后，解耦了原来各个部分强耦合的关系。

#### 四、实现WebView的HttpDns和本地图片、文件的缓存

前面介绍了整个客户端H5服务的架构，我们就基于这个架构实现一个技术优化需求，那就是提升H5页面的加载速度。我们先来看下，H5页面的网络请求可能包括以下几部分

1. Html和js、css等文件的加载
2. 图片等素材的下载
3. 异步网络请求，比如一些Ajax请求

在这些网络请求中，我们面临一些问题：

1. 图片在Native中下载过，点进去H5时还得再下一遍，图片无法复用
2. Ajax异步网络请求不能添加Header或者其他的一些处理逻辑（Webview只能在loadUrl的时候添加header）
3. Html和js、css反复下载，不能进行可靠的管理
4. 在一些网络环境中DNS服务并不可靠，DNS结果可能拿不到或者被劫持（笔者在项目中就跟过几例在浙江移动网络中文件被劫持成其他文件的情况）。

要解决这些问题，我们就需要实现由客户端代理WebView的网络请求，还需要实现WebView加载客户端本地缓存的能力。 WebViewClient类中提供了shouldInterceptRequest的方法。

##### 1.shouldInterceptRequest接口

```java
public WebResourceResponse shouldInterceptRequest(WebView view, final WebResourceRequest webResourceRequest)
```

通过复写这个方法我们可以拦截浏览器的资源请求，返回指定的内容。这个接口在文档中是这么描述的

> Notify the host application of a resource request and allow the application to return the data. If the return value is null, the WebView will continue to load the resource as usual. Otherwise, the return response and data will be used. This callback is invoked for a variety of URL schemes (e.g., http(s):, data:, file:, etc.), not only those schemes which send requests over the network. This is not called for javascript: URLs, blob: URLs, or for assets accessed via file:///android_asset/ or file:///android_res/ URLs. In the case of redirects, this is only called for the initial resource URL, not any subsequent redirect URLs.

这里面有几点需要注意的

**1. 这个接口不只是http或者https的请求才出发的，也有可能是data或者file协议，但是不包括javascript、file:///android_asset或者file:///android_res。所以拦截的时候我们要注意只拦截网络请求。 2. 返回值为空时浏览器会按原有的逻辑执行请求，否则就使用指定的数据。 3. 重定向的请求只有第一个链接会调用这个接口，后续跳转链接不会。**

那么这个接口怎么使用呢，看看这个接口的参数和返回值就知道了。这个接口的参数 WebResourceRequest 的定义：

```java
public interface WebResourceRequest {

    /**
     * 请求的url
     */
    Uri getUrl();

    /**
     * 返回该请求是不是主Frame发出的
     */
    boolean isForMainFrame();

    /**
     * 该请求是否是服务端重定向的结果
     */
    boolean isRedirect();

    /**
     * 该请求是否与用户的手势有关，比如点击等
     */
    boolean hasGesture();

    /**
     * 网络的请求Method，比如GET或者POST
     */
    String getMethod();

    /**
     * 网络请求的header
     */
    Map<String, String> getRequestHeaders();
}
```

这里面包含了请求的基本信息，再来看看返回值WebResourceResponse的相关属性

```java
public class WebResourceResponse {
    private String mMimeType;//资源的MIME类型，比如text/html
    private String mEncoding;//response的编码格式，比如utf-8
    private int mStatusCode;//http状态码
    private String mReasonPhrase;//状态码描述与，比如200对应的是“OK”，这个值不能为空
    private Map<String, String> mResponseHeaders;//response的header
    private InputStream mInputStream;//输入流
}
```

##### 2.实现代理资源请求和HttpDns

由于国内Dns并不是十分可靠，很多公司都会考虑接入HttpDns服务，我们在网络层中已经实现了HttpDns相关功能，是通过OkHttp的dns接口实现的，有兴趣的可以看一下这两篇文章 [OkHttp接入HttpDNS，最佳实践](https://www.jianshu.com/p/6bd131de81d3) [okhttp源码解析（五）：代理和DNS](https://blog.csdn.net/u011315960/article/details/81285154) 相关原理在这里就不再赘述，我们重点关心一下如何拦截WebView的请求，并使用OkHttp做网络请求

```java
    Request.Builder builder = new Request.Builder();
    //构造请求
    builder.url(url).method(webResourceRequest.getMethod(), null);
    Map<String, String> requestHeaders = webResourceRequest.getRequestHeaders();
    if (Lang.isNotEmpty(requestHeaders)) {
        for (Map.Entry<String, String> entry : requestHeaders.entrySet()) {
            builder.addHeader(entry.getKey(), entry.getValue());
        }
    }
    Call synCall = mClient.newCall(builder.build());
    okhttp3.Response response = synCall.execute();
    if (response.body() != null) {
        ResponseBody body = response.body();
        Map<String, String> map = new HashMap<>();
        for (int i = 0; i < response.headers().size(); i++) {
        //相应体的header
            map.put(response.headers().name(i), response.headers().value(i));
        }
        //MIME类型
        String mimeType = MimeTypeMap.getSingleton().getMimeTypeFromExtension(MimeTypeMap.getFileExtensionFromUrl(url));
        String contentType = response.headers().get("Content-Type");
        String encoding = "utf-8";
        //获取ContentType和编码格式
        if (contentType != null && !"".equals(contentType)) {
            if (contentType.contains(";")) {
                String[] args = contentType.split(";");
                mimeType = args[0];
                String[] args2 = args[1].trim().split("=");
                if (args.length == 2 && args2[0].trim().toLowerCase().equals("charset")) {
                    encoding = args2[1].trim();
                }
            } else {
                mimeType = contentType;
            }
        }
        WebResourceResponse webResourceResponse = new WebResourceResponse(mimeType, encoding, body.byteStream());
        String message = response.message();
        int code = response.code();
        if (TextUtils.isEmpty(message) && code == 200) {
            //message不能为空
            message = "OK";
        }
        webResourceResponse.setStatusCodeAndReasonPhrase(code, message);
        webResourceResponse.setResponseHeaders(map);
```

这里面需要注意的是对重定向的处理，笔者在实际项目中使用时，如果有重定向的code返回，OkHttp中是可以直接处理重定向的结果，并且返回重定向后网络请求的结果。但是把这个结果返回给给WebView后，WebView并不知道经过302跳转了，所以会将当前的响应与重定向前的url关联，导致页面加载时可能出现错误，所以需要把302的情况交回给WebView进行处理。在OkHttp中把重定向的follow设置为false:`okClientBuilder.followRedirects(false);` 这样，就可以代理网络请求。结合我们基于OkHttp做的HttpDns，就可以解决WebView的请求Dns的问题。 此外，如有应用对Cookie有需求，可以自行设置Cookie的请求，相关设置可以参考[有赞技术团队的相关文章](https://tech.youzan.com/you-zan-webview-goldwing-three/)

##### 3.加载本地缓存

前面介绍过我们把H5页面的资源分成三个部分：

1. Html和js、css等文件的加载
2. 图片等素材的下载
3. 异步网络请求，比如一些Ajax请求

第3点我们采用的是由客户端对一些数据进行预缓存和预请求，在用户打开页面的时候，再将缓存数据通过jsBridge传递到H5中，提高页面的加载速度。 对于1和2可以由WebView自带的缓存，但是缺点比较明显，首先，客户端很难操作WebView的缓存，没办通过WebView的缓存进行一些预下载和预更新的逻辑，造成首次进入时网络请求的时间过长。其次，图片在Native下载过，在WebView可能又得下载一遍，比如文章的封面图，在列表中加载过一次，在H5中又需要下载一次。所以我们考虑在Native中做代码包管理和共享Native的图片缓存。这些资源都以文件的形式存放在用户的手机上。同样通过shouldInterceptRequest接口实现：

```java
fileInputStream = new FileInputStream(file);
String type = "text/html";
if (path.contains(".css")) {
    type = "text/css";
} else if (path.contains(".js")) {
    type = "application/javascript";
}
WebResourceResponse response = new WebResourceResponse(type, "utf-8", fileInputStream);
Map<String, String> map = new HashMap<>();
map.put("Content-Type", type);
//跨域响应头，否则可能会有跨域无法访问的问题
map.put("Access-Control-Allow-Origin", "*");
response.setResponseHeaders(map);
return response;
```

把本地文件流作为WebResourceResponse的输入流，这样就把网络请求替代问本地文件。基于此，我们也实现了本地包的管理和图片的缓存。而这一切对于H5页面来说甚至是毫无感知的。 优化完成后，懂球帝客户端打开文章页和加载封面图，几乎是秒开的速度



![img](https://user-gold-cdn.xitu.io/2019/12/7/16ee02feb67baaff?imageslim)



#### 四、总结与展望

1、笔者首先对H5相关的业务进行了抽象，梳理了这其中的信息流和调用关系，采用策略模式重构了客户端的WebView页面，依赖于接口而不是原来的依赖于具体实现。
 2、基于策略模式的接口，笔者实现了对于H5的网络资源请求的代理，为H5页面的网络请求实现HttpDns的功能，解决线上的DNS问题。
 3、基于策略模式的接口，笔者实现H5包和图片的本地缓存。
 通过以上几个优化，懂球帝的Android客户端中与WebView相关的业务都得到了极大的优化，十几个页面中重复的数千行代码和逻辑得到了重新的梳理，有些页面甚至简化了上千行代码。基于策略模式，WebView的功能扩展也得到了良好的扩展性和约束。至此我们对于WebView页面的优化也告一段落，当然后面我们还会持续做一些优化，有几个方向
 1、目前包缓存只在部分页面应用了，未来可以扩展到更多的页面和更细粒度的文件上
 2、提高WebView的一致性，更换第三方内核
 3、对WebView中的音视频进行优化
 ......

#### 参考文献

【1】[如何设计一个优雅健壮的Android WebView？（下）](https://iluhcm.com/2018/02/27/design-an-elegant-and-powerful-android-webview-part-two/)
 【2】[有赞webview加速平台探索与建设（三）——html加速](https://tech.youzan.com/you-zan-webview-goldwing-three/)
 【3】《研磨设计模式》，清华大学出版社，陈臣，王斌
 【4】[Android Developers WebResourceResponse](https://developer.android.com/reference/android/webkit/WebResourceResponse.html)
 【5】[Android：手把手教你构建 全面的WebView 缓存机制 & 资源加载方案](https://www.jianshu.com/p/5e7075f4875f)


链接：https://juejin.im/post/5deb7e9751882512756e82d1来源：掘金著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。