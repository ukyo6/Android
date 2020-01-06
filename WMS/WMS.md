上一篇文章[【理清Activity、View及Window之间关系】](https://link.jianshu.com?t=http://blog.csdn.net/huachao1001/article/details/51866287)我们大致知道了`Window`的绘制过程，但是比较笼统，本文主要介绍`Window`对象与（后面缩写为`WMS`）之间是如何通信。毫无疑问，肯定是通过`IPC`（`Binder`机制），这点肯定都知道，但是我们要学习是的是，哪些类参与了`IPC`调用过程。另外，本文没有研究源码，而是通过阅读其他研究源码的文章，然后总结出来，以更容易理解的方式展示。本文设计到的相关资料在文章最后一一列出。

## 1 Window添加的大致过程

`Window`的添加过程需要通过`WindowManager`的`addView`来实现，`WindowManager`是一个接口，真正的实现是`WindowManagerImpl`。而`WindowManagerImpl`全部是转移给`WindowManagerGlobal`来处理，`WindowManagerImpl`这种工作模式是典型的桥接模式。`WindowManagerImpl`内三大操作过程如下：

```java
public void addView(View view,ViewGroup.LayoutParams params){
    mGlobal.addView(view,params,mDisplay,mParantWindow);
}

public void updateViewLayout(View view,ViewGroup.LayoutParams params){
    mGlobal.updateViewLayout(view,params);
}

public void removeView(View view){
    mGlobal.removeView(view,false);
}
```

从`WindowManagerGlobal`名称可以看出，它是一个全局的`WindowManager`，其内部维护如下几个列表：

```java
private final ArrayList<View> mViews = new ArrayList<View>();
private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>()
private final ArrayList<WindowManager.LayoutParams> mParams = new ArrayList<WindowManager.layoutParams>();
private final ArraySet<View> mDyingViews = new ArraySet<View>();
```

其中：

> - **mViews**：存储所有`Window`所对应的`View`
> - **mRoots**：存储的是所有`Window`所对应的`ViewRootImpl`
> - **mParams**：存储的是所有`Window`所对应的布局参数
> - **mDyingView**：存储了那些正在被删除的`View`对象，或者说是那些已经调用了`removeView`方法但是删除操作还未完成的`Window`对象。

`addView`中通过如下方式将`Window`的一系列对象添加到列表中：

```java
root=new ViewRootImpl(view.getContext(),display);
view.setLayoutParams(wparams);

mViews.add(view);
mRoots.add(root);
mParams.add(wparams);
```

可以看到，到目前为止，只是把相应的对象存放到`ArrayList`列表中。后面还需要将`View`给显示出来。绘制`View`需要通过`ViewRootImpl`的`setView`方法来实现。在`setView`内部是通过`requestLayout`来完成一部刷新请求的。

```java
public void requestLayout(){
    if(!mHandlingLayoutInlayoutRequest){
        checkThread();
        mLayoutRequested=true;
        scheduleTraversals(); //界面绘制的入口
    }
}
```

其中`scheduleTraversals`是`View`绘制的入口！

下面我们看看`ViewRootImpl`是内部机制。

## 2 ViewRootImpl内部机制

`ViewRootImpl`用于管理窗口的根`View`，并和`WMS`进行交互。`ViewRootImpl`中有一个内部类： `W`，以及另一个内部类：`ViewRootHandler`。

> - `W`继承自`IWindow.Stub`。是一个`Binder`对象，用于接收`WMS`的各种消息， 如按键消息， 触摸消息等。
> - `ViewRootHandler`，是`Handler`的子类， `W`会通过`Looper`把消息传递给`ViewRootHandler`。

`ViewRootImpl`有一个`W`类型的成员`mWindow`，`ViewRootImpl`在构造函数中创建一个`W`的实例并赋值给`mWindow`。

在`ViewRootImpl`的`setView`方法（此方法运行在`UI`线程）中，会通过`IPC`的方式跨进程向`WMS`发起一个远程调用，从而将`DecorView`最终添加到`Window`上，在这个过程中，`ViewRootImpl`、`DecorView`和`WMS`会彼此向关联.

另外，`WMS`有时也需要向`ViewRootImpl`发送远程请求，比如，点击事件是由用户的触摸行为所产生的，因此它必须要通过硬件来捕获，跟硬件之间的交互自然是Android系统自己把握，Android系统将点击事件交给`WMS`来处理。`WMS`通过远程调用将事件发送给`ViewRootImpl`，在`ViewRootImpl`中，有一个方法，叫做`dispatchInputEvent`，最终将事件传递给`DecorView`。

## 3 Window与WMS之间的双向通信

接下来，由`WindowSession`来完成最后的`Window`添加过程。`mWindowSession`本身是一个`IWindowSession`类型对象，通过内部代理类`Proxy`访问远程`Session`类（`Binder`机制） 。在`Session`内部通过`WMS`来实现`Window`的添加。如此一来`Window`的添加请求就交给了`WMS`去处理了，如下图所示：

![ViewRootImpl与WmS之间通信](http://img.blog.csdn.net/20160726160539060)

ViewRootImpl与WmS之间通信

`WindowManagerService`内部为每个应用保留一个单独的`Session`，如下图所示：

![img](https:////upload-images.jianshu.io/upload_images/2154124-5fc46fcdd4b916dc?imageMogr2/auto-orient/strip|imageView2/2/w/873/format/webp)

每个应用对应一个Session

前面我们说过，每个`Window`对应一个`ViewRootImpl`及一个`View Tree`。也就是说，每个`Activity`对应的`Window`（一个或多个）内通过其内置的`ViewRootImpl`完成向`WMS`的请求过程。

一个应用中的所有`Activity`共用一个`Session`，一个`Window`在`WMS`内部对应一个`WindowState`，`WindowState`维护窗口的状态以及根据适当的机制来调整窗口的状态。

如果一个`Activity`多个`Window`，如对话框、`Popup`类型、或者通过`ViewManager`将`View`直接加入`WMS`，等等。在这些情况下，一个`Activity`就会创建多个`Window`，相应的`WMS`中也会对应多个`WindowState`，如下图所示：

![img](https:////upload-images.jianshu.io/upload_images/2154124-28081abd74c4f4bf?imageMogr2/auto-orient/strip|imageView2/2/w/704/format/webp)

多个Window情况下对应关系

## 4 WMS控制窗口的显示



 **以下内容来自老罗的的博客，后面附有资料链接。**

`WMS`服务大致按照以下方式来控制哪些窗口需要显示的以及要显在哪里：

1. 每一个`Activity`窗口的大小都等于屏幕的大小，因此，只要对每一个`Activity`窗口设置一个不同的`Z`轴位置，然后就可以使得位于最上面的，即当前被激活的`Activity`窗口，才是可见的。
2. 每一个子窗口的Z轴位置都比它的父窗口大，但是大小要比父窗口小，这时候`Activity`窗口及其所弹出的子窗口都可以同时显示出来。
3. 对于非全屏`Activity`窗口来说，它会在屏幕的上方留出一块区域，用来显示状态栏。这块留出来的区域称对于屏幕来说，称为装饰区（`decoration`），而对于`Activity`窗口来说，称为内容边衬区（`Content Inset`）。
4. 输入法窗口只有在需要的时候才会出现，它同样是出现在屏幕的装饰区或者说`Activity`窗口的内容边衬区的。
5. 对于壁纸窗口，它出现需要壁纸的`Activity`窗口的下方，这时候要求`Activity`窗口是半透明的，这样就可以将它后面的壁纸窗口一同显示出来。
6. 两个`Activity`窗口在切换过程，实际上就是前一个窗口显示退出动画而后一个窗口显示开始动画的过程，而在动画的显示过程，窗口的大小会有一个变化的过程，这样就导致前后两个`Activity`窗口的大小不再都等于屏幕的大小，因而它们就有可能同时都处于可见的状态。事实上，`Activity`窗口的切换过程是相当复杂的，因为即将要显示的`Activity`窗口可能还会被设置一个启动窗口（`Starting Window`）。一个被设置了启动窗口的`Activity`窗口要等到它的启动窗口显示了之后才可以显示出来。

## 5. 为什么没法获取宽高

![img](https://upload-images.jianshu.io/upload_images/3985563-e773ab2cb83ad214.png?imageMogr2/auto-orient/strip|imageView2/2/w/690/format/webp)



1. Window.setContentView() 

2. PhoneWindow.setContentView ()

   ```java
   	@Override
       public void setContentView(int layoutResID) {
           if (mContentParent == null) {
               installDecor(); //初始化decorView
           } else {
               mContentParent.removeAllViews();
           }
           mLayoutInflater.inflate(layoutResID, mContentParent); //把我们的布局inflate到android.R.id.content中
           final Callback cb = getCallback();
           if (cb != null && !isDestroyed()) {
               cb.onContentChanged();
           }
   	}
   ```

   

3. ActivityThread.handleResumeActivity()

   ```java
       public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,String reason) {
               //先调用Activity.onResume
               final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
                       
               //再调用wm.addView(decorView)         
               r.window = r.activity.getWindow();
               View decor = r.window.getDecorView();
               wm.addView(decor, l);
       }
   ```

4. WindowManagerGlobal.addView()

   ```java
       public void addView(View view, ViewGroup.LayoutParams params,Display display, Window parentWindow) {
               ViewRootImpl root;
               root = new ViewRootImpl(view.getContext(), display);
               root.setView(view, wparams, panelParentView);
       }
   ```

5. ViewRootImpl.setView(view)

   ```java
       public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
           synchronized (this) {
               if (mView == null) {
                   requestLayout();
               }
           }
       }                 
   
       @Override
       public void requestLayout() {
           if (!mHandlingLayoutInLayoutRequest) {
               checkThread();
               mLayoutRequested = true;
               scheduleTraversals(); //开始绘制界面
           }
       }
   ```

   

