
**本次glide的分析是基于3.7版本**

glide加载图片的调用很简单， Glide.with(activity/fragment/application).load("https://xxxxx.png").into(imageView)。  
但是使用不当，会导致内存泄漏。要分析为什么会导致内存泄漏，那就必须得看源码了。

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
看一下方法签名，使用glide开始一个加载，并且绑定到给定的activity的生命周期。  既然会绑定activity的生命周期，那肯定是要做些什么。大多数开源库一般情况下会使用生命周期去释放资源，避免内存泄漏。  

**下面看下glide是如何把绑定到activity的生命周期的**

`RequestManagerRetriever.get()`返回RequestManagerRetriever的单例对象，主要看`retriever.get(activity)`
```

public class RequestManagerRetriever{

    public RequestManager get(FragmentActivity activity) {
        // 先不看非主线程的情况
        if (Util.isOnBackgroundThread()) {
            return get(activity.getApplicationContext());
        } else {
            assertNotDestroyed(activity);
            // 获取了activity的fragment管理器
            FragmentManager fm = activity.getSupportFragmentManager();
            // 看方法的意思应该是获取一个fragment，并且传入了activity的fragment管理器
            // 可以猜想一下了，应该就是创建一个不可见的没有UI的，fragment添加到activity的fragment管理器里面
            //这样 就能监听到activity的生命周期了 ，这是个套路比较常见。例如lifecycle的实现，也是这样
            return supportFragmentGet(activity, fm);
        }
    }
    
    RequestManager supportFragmentGet(Context context, FragmentManager fm) {
        // 创建一个fragment
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
    
    SupportRequestManagerFragment getSupportRequestManagerFragment(final FragmentManager fm) {
        SupportRequestManagerFragment current = (SupportRequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
        if (current == null) {
            current = pendingSupportRequestManagerFragments.get(fm);
            if (current == null) {
                // 创建SupportRequestManagerFragment 并且 添加到FragmentManager
                current = new SupportRequestManagerFragment();
                pendingSupportRequestManagerFragments.put(fm, current);
                fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();
                handler.obtainMessage(ID_REMOVE_SUPPORT_FRAGMENT_MANAGER, fm).sendToTarget();
            }
        }
        return current;
    }
    ...
}
```

再来看一下那个不可见的fragment长什么样子，是怎么和glide关联起生命周期的。
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
    

class ActivityFragmentLifecycle implements Lifecycle {
    private final Set<LifecycleListener> lifecycleListeners = Collections.newSetFromMap(new WeakHashMap<LifecycleListener, Boolean>());
    private boolean isStarted;
    private boolean isDestroyed;
    
     @Override
    public void addListener(LifecycleListener listener) {
        lifecycleListeners.add(listener);

        if (isDestroyed) {
            listener.onDestroy();
        } else if (isStarted) {
            listener.onStart();
        } else {
            listener.onStop();
        }
    }

    void onStart() {
        isStarted = true;
        for (LifecycleListener lifecycleListener : Util.getSnapshot(lifecycleListeners)) {
            lifecycleListener.onStart();
        }
    }

    void onStop() {
        isStarted = false;
        for (LifecycleListener lifecycleListener : Util.getSnapshot(lifecycleListeners)) {
            lifecycleListener.onStop();
        }
    }

    void onDestroy() {
        isDestroyed = true;
        for (LifecycleListener lifecycleListener : Util.getSnapshot(lifecycleListeners)) {
            lifecycleListener.onDestroy();
        }
    }
}


 
public class RequestManager implements LifecycleListener{
            RequestManager(Context context, final Lifecycle lifecycle, RequestManagerTreeNode treeNode,
                RequestTracker requestTracker, ConnectivityMonitorFactory factory) {
            this.context = context.getApplicationContext();
            this.lifecycle = lifecycle;
            this.treeNode = treeNode;
            this.requestTracker = requestTracker;
            // 创建glide实例，glide是个单例
            this.glide = Glide.get(context);
            this.optionsApplier = new OptionsApplier();
    
            ConnectivityMonitor connectivityMonitor = factory.build(context,
                    new RequestManagerConnectivityListener(requestTracker));
             // 将RequestManager添加到fragemnt里面的lifecycle里，回调生命周期
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


### RequestManager 如何根据生命周期管理资源
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


![](https://user-gold-cdn.xitu.io/2019/9/17/16d3dee1b7f17e93?w=317&h=555&f=png&s=15011)

EngineRunnable 通过线程池异步执行，所以在请求期间如果GenricRequest不根据生命周期释放引用，将会导致内存泄漏。

### 既然glide有了资源处理的代码，为什么会导致内存泄漏

glide.with有如下重载方法 可以传入 context activity fragment
```
    public static RequestManager with(Context context) {
        RequestManagerRetriever retriever = RequestManagerRetriever.get();
        return retriever.get(context);
    }
    
    public static RequestManager with(Activity activity) {
        RequestManagerRetriever retriever = RequestManagerRetriever.get();
        return retriever.get(activity);
    }
    
    public static RequestManager with(FragmentActivity activity) {
        RequestManagerRetriever retriever = RequestManagerRetriever.get();
        return retriever.get(activity);
    }
    
public class RequestManagerRetriever{
        
    public RequestManager get(android.app.Fragment fragment) {
        if (fragment.getActivity() == null) {
            throw new IllegalArgumentException("You cannot start a load on a fragment before it is attached");
        }
        if (Util.isOnBackgroundThread() || Build.VERSION.SDK_INT < Build.VERSION_CODES.JELLY_BEAN_MR1) {
            return get(fragment.getActivity().getApplicationContext());
        } else {
            // 如果传入的是fragment ,使用的是fragment.getChildFragmentManager 绑定的就是fragment的生命周期
            android.app.FragmentManager fm = fragment.getChildFragmentManager();
            return fragmentGet(fragment.getActivity(), fm);
        }
    }
    
    public RequestManager get(Context context) {
        if (context == null) {
            throw new IllegalArgumentException("You cannot start a load on a null Context");
        } else if (Util.isOnMainThread() && !(context instanceof Application)) {
            if (context instanceof FragmentActivity) {
                return get((FragmentActivity) context);
            } else if (context instanceof Activity) {
                return get((Activity) context);
            } else if (context instanceof ContextWrapper) {
                return get(((ContextWrapper) context).getBaseContext());
            }
        }
        // 如果使用的是 Contxet 使用的就是application的生命周期
        return getApplicationManager(context);
    }
    
}
    
```
**所以内存泄漏的原因就是没正确使用Glide.with()入参导致的**  
当加载发生在Fragment的时候，应该传入Fragment。如果传入Activity，将会导致Fragment泄漏。  
当传入Application的时候，将会根据application的生命周期来回调。也就起不到释放资源的作用了。  
不过我们可以通过手动调用Glide.clear() 来释放资源


### 总结
**RequestManager** 实现LifecycleListener 接口 ，管理加载图片的请求。  
**SupportRequestManagerFragment** 一个没有界面的fragment，实例化的时候会创建ActivityFragmentLifecycle，将这个fragment的生命周期回调出来。RequestManager实例化的时候会绑定到ActivityFragmentLifecycle，这样就可以根据fragment的生命周期启动暂停和回收请求了。

**Glide.with**的作用就是 创建一个不可见的fragment，绑定到activity或者fragment的生命周期，并且实例化一个RequestManager对象接受生命周期回调。

**调用glide.with()的时候需要正确传入参数才能避免内存泄漏**


---

## RequestManager.load()
Glide里面用了大量的泛型和工厂方法，介绍 RequestManager.load()方法的时候 我们得先介绍几个概念，不然后面的代码越看越懵
  
先看一个glide加载的流程图



看下glide的实例化，glide对象是在 Glide.with()方法内 创建RequestManager对象的时候创建的。可以向上查找相关代码。



```

public class Glide{
    private final GenericLoaderFactory loaderFactory;
    private final DataLoadProviderRegistry dataLoadProviderRegistry;
    private final TranscoderRegistry transcoderRegistry = new TranscoderRegistry();
    ...
    
    Glide(Engine engine, MemoryCache memoryCache, BitmapPool bitmapPool, Context context, DecodeFormat decodeFormat) {
        this.engine = engine;
        this.bitmapPool = bitmapPool;
        this.memoryCache = memoryCache;
        this.decodeFormat = decodeFormat;
        loaderFactory = new GenericLoaderFactory(context);
        mainHandler = new Handler(Looper.getMainLooper());
        bitmapPreFiller = new BitmapPreFiller(memoryCache, bitmapPool, decodeFormat);

        dataLoadProviderRegistry = new DataLoadProviderRegistry();

        StreamBitmapDataLoadProvider streamBitmapLoadProvider =
                new StreamBitmapDataLoadProvider(bitmapPool, decodeFormat);
        dataLoadProviderRegistry.register(InputStream.class, Bitmap.class, streamBitmapLoadProvider);

        FileDescriptorBitmapDataLoadProvider fileDescriptorLoadProvider =
                new FileDescriptorBitmapDataLoadProvider(bitmapPool, decodeFormat);
        dataLoadProviderRegistry.register(ParcelFileDescriptor.class, Bitmap.class, fileDescriptorLoadProvider);

        ImageVideoDataLoadProvider imageVideoDataLoadProvider =
                new ImageVideoDataLoadProvider(streamBitmapLoadProvider, fileDescriptorLoadProvider);
        dataLoadProviderRegistry.register(ImageVideoWrapper.class, Bitmap.class, imageVideoDataLoadProvider);

        GifDrawableLoadProvider gifDrawableLoadProvider =
                new GifDrawableLoadProvider(context, bitmapPool);
        dataLoadProviderRegistry.register(InputStream.class, GifDrawable.class, gifDrawableLoadProvider);

        dataLoadProviderRegistry.register(ImageVideoWrapper.class, GifBitmapWrapper.class,
                new ImageVideoGifDrawableLoadProvider(imageVideoDataLoadProvider, gifDrawableLoadProvider, bitmapPool));

        dataLoadProviderRegistry.register(InputStream.class, File.class, new StreamFileDataLoadProvider());

        register(File.class, ParcelFileDescriptor.class, new FileDescriptorFileLoader.Factory());
        register(File.class, InputStream.class, new StreamFileLoader.Factory());
        register(int.class, ParcelFileDescriptor.class, new FileDescriptorResourceLoader.Factory());
        register(int.class, InputStream.class, new StreamResourceLoader.Factory());
        register(Integer.class, ParcelFileDescriptor.class, new FileDescriptorResourceLoader.Factory());
        register(Integer.class, InputStream.class, new StreamResourceLoader.Factory());
        register(String.class, ParcelFileDescriptor.class, new FileDescriptorStringLoader.Factory());
        register(String.class, InputStream.class, new StreamStringLoader.Factory());
        register(Uri.class, ParcelFileDescriptor.class, new FileDescriptorUriLoader.Factory());
        register(Uri.class, InputStream.class, new StreamUriLoader.Factory());
        register(URL.class, InputStream.class, new StreamUrlLoader.Factory());
        register(GlideUrl.class, InputStream.class, new HttpUrlGlideUrlLoader.Factory());
        register(byte[].class, InputStream.class, new StreamByteArrayLoader.Factory());

        transcoderRegistry.register(Bitmap.class, GlideBitmapDrawable.class,
                new GlideBitmapDrawableTranscoder(context.getResources(), bitmapPool));
        transcoderRegistry.register(GifBitmapWrapper.class, GlideDrawable.class,
                new GifBitmapWrapperDrawableTranscoder(
                        new GlideBitmapDrawableTranscoder(context.getResources(), bitmapPool)));

        bitmapCenterCrop = new CenterCrop(bitmapPool);
        drawableCenterCrop = new GifBitmapWrapperTransformation(bitmapPool, bitmapCenterCrop);

        bitmapFitCenter = new FitCenter(bitmapPool);
        drawableFitCenter = new GifBitmapWrapperTransformation(bitmapPool, bitmapFitCenter);
    }
    
}
```






```
    public DrawableTypeRequest<String> load(String string) {
        return (DrawableTypeRequest<String>) fromString().load(string);
    }
    
    public DrawableTypeRequest<String> fromString() {
        return loadGeneric(String.class);
    }
    
    private <T> DrawableTypeRequest<T> loadGeneric(Class<T> modelClass) {
        ModelLoader<T, InputStream> streamModelLoader = Glide.buildStreamModelLoader(modelClass, context);
        ModelLoader<T, ParcelFileDescriptor> fileDescriptorModelLoader =
                Glide.buildFileDescriptorModelLoader(modelClass, context);
        if (modelClass != null && streamModelLoader == null && fileDescriptorModelLoader == null) {
            throw new IllegalArgumentException("Unknown type " + modelClass + ". You must provide a Model of a type for"
                    + " which there is a registered ModelLoader, if you are using a custom model, you must first call"
                    + " Glide#register with a ModelLoaderFactory for your custom model class");
        }

        return optionsApplier.apply(
                new DrawableTypeRequest<T>(modelClass, streamModelLoader, fileDescriptorModelLoader, context,
                        glide, requestTracker, lifecycle, optionsApplier));
    }
```


