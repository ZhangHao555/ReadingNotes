
**本次glide的分析是基于3.7版本**

glide加载图片的调用很简单， Glide.with(activity/fragment/application).load("https://xxxxx.png").into(imageView)。简单的调用下面却隐藏了比较复杂的逻辑，我们一步步来查看源码看看为什么会导致内存泄漏。

## Glide.with()
```
    /**
     * Begin a load with Glide that will be tied to the given {@link android.app.Activity}'s lifecycle and that uses the
     * given {@link Activity}'s default options.
     *
     * @param activity The activity to use.
     * @return A RequestManager for the given activity that can be used to start a load.
     */
    public static RequestManager with(Activity activity) {
        RequestManagerRetriever retriever = RequestManagerRetriever.get();
        return retriever.get(activity);
    }
```
看一下方法签名，使用glide开始一个加载，并且绑定到给定的activity的生命周期。  
glide的一个特色就是会根据activity的状态开始、暂停加载，及时释放避免内存泄漏。（但是如果glide.with()的参数是application时，就需要手动调用Gldie.clear()了） 

`RequestManagerRetriever.get()`返回RequestManagerRetriever的单例对象，主要看`retriever.get(activity)`
```
    public RequestManager get(FragmentActivity activity) {
        if (Util.isOnBackgroundThread()) {
            return get(activity.getApplicationContext());
        } else {
            assertNotDestroyed(activity);
            FragmentManager fm = activity.getSupportFragmentManager();
            // 主要看这里，获取一个不可见的fragment，并且添加的activity的FragmentManager里面，用于监听生命周期
            return supportFragmentGet(activity, fm);
        }
    }
```

```
    RequestManager supportFragmentGet(Context context, FragmentManager fm) {
        // 创建一个不可见的fragment
        SupportRequestManagerFragment current = getSupportRequestManagerFragment(fm);
        RequestManager requestManager = current.getRequestManager();
        if (requestManager == null) {
            // 创建一个RequestManager，并且绑定SupportRequestManagerFragment的生命周期
            //RequestManager 顾名思义，请求管理器，通过生命周期回调管理了glide的加载暂停和资源回收
            requestManager = new RequestManager(context, current.getLifecycle(), current.getRequestManagerTreeNode());
            current.setRequestManager(requestManager);
        }
        return requestManager;
    }
```

看一下那个不可见的fragment长什么样子，是怎么和glide关联起生命周期的。
```
public class SupportRequestManagerFragment extends Fragment {
    private RequestManager requestManager;
    //生命周期回调
    private final ActivityFragmentLifecycle lifecycle;
    private final RequestManagerTreeNode requestManagerTreeNode =
            new SupportFragmentRequestManagerTreeNode();
    private final HashSet<SupportRequestManagerFragment> childRequestManagerFragments =
        new HashSet<SupportRequestManagerFragment>();
    private SupportRequestManagerFragment rootRequestManagerFragment;

    public SupportRequestManagerFragment() {
        this(new ActivityFragmentLifecycle());
    }
    
    @Override
    public void onStart() {
        super.onStart();
        lifecycle.onStart();
    }

    @Override
    public void onStop() {
        super.onStop();
        lifecycle.onStop();
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        lifecycle.onDestroy();
    }

    @Override
    public void onLowMemory() {
        super.onLowMemory();
        // If an activity is re-created, onLowMemory may be called before a manager is ever set.
        // See #329.
        if (requestManager != null) {
            requestManager.onLowMemory();
        }
    }
    ...
    public class RequestManager implements LifecycleListener{
            RequestManager(Context context, final Lifecycle lifecycle, RequestManagerTreeNode treeNode,
                RequestTracker requestTracker, ConnectivityMonitorFactory factory) {
            this.context = context.getApplicationContext();
            this.lifecycle = lifecycle;
            this.treeNode = treeNode;
            this.requestTracker = requestTracker;
            this.glide = Glide.get(context);
            this.optionsApplier = new OptionsApplier();
    
            ConnectivityMonitor connectivityMonitor = factory.build(context,
                    new RequestManagerConnectivityListener(requestTracker));
             // 将RequestManager添加到回调里
            if (Util.isOnBackgroundThread()) {
                new Handler(Looper.getMainLooper()).post(new Runnable() {
                    @Override
                    public void run() {
                        lifecycle.addListener(RequestManager.this);
                    }
                });
            } else {
                lifecycle.addListener(this);
            }
            lifecycle.addListener(connectivityMonitor);
        }
    }
```

### 时序图

![](https://user-gold-cdn.xitu.io/2019/9/15/16d354f1d1ed1bdc?w=1477&h=674&f=png&s=81237)

### 总结
**RequestManager** 实现LifecycleListener 接口 ，管理加载图片的请求。  
**SupportRequestManagerFragment** 一个没有界面的fragment，实例化的时候会创建ActivityFragmentLifecycle，将这个fragment的生命周期回调出来。RequestManager实例化的时候会绑定到ActivityFragmentLifecycle，这样就可以根据fragment的生命周期启动暂停和回收请求了。

**Glide.with**的作用就是 创建一个不可见的fragment，并且绑定到activity或者fragment的生命周期，并且实例化一个RequestManager对象接受生命周期回调。


## RequestManager 如何根据生命周期管理资源
```
public class RequestManager{
    //只贴出与生命周期有关的代码
    
    @Override
    public void onStart() {
        resumeRequests();
    }
    
    public void resumeRequests() {
        Util.assertMainThread();
        requestTracker.resumeRequests();
    }
    
    @Override
    public void onStop() {
        pauseRequests();
    }
    
    public void pauseRequests() {
        Util.assertMainThread();
        requestTracker.pauseRequests();
    }
    
    @Override
    public void onDestroy() {
        requestTracker.clearRequests();
    }
    
    ...
}
RequestManager 只是通知回调，具体的开始暂停逻辑在 RequestTracker里面


public class RequestTracker {

    public void resumeRequests() {
        isPaused = false;
        for (Request request : Util.getSnapshot(requests)) {
            if (!request.isComplete() && !request.isCancelled() && !request.isRunning()) {
                request.begin();
            }
        }
        pendingRequests.clear();
    }
    
    public void pauseRequests() {
        isPaused = true;
        for (Request request : Util.getSnapshot(requests)) {
            if (request.isRunning()) {
                request.pause();
                pendingRequests.add(request);
            }
        }
    }
    
    public void clearRequests() {
        for (Request request : Util.getSnapshot(requests)) {
            request.clear();
        }
        pendingRequests.clear();
    }
    ...
}
```

**以下是一个请求的引用链**

![](https://user-gold-cdn.xitu.io/2019/9/16/16d387f2510943c3?w=591&h=473&f=png&s=21018)

EngineRunnable 通过线程池异步执行，所以在请求期间如果GenricRequest不根据生命周期释放引用，将会导致内存泄漏。



