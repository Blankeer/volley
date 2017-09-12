# Volley
## Demo
```
    RequestQueue mQueue = Volley.newRequestQueue(this);
    StringRequest request = new StringRequest(Request.Method.GET, "http://www.baidu.com",
              new Response.Listener<String>() {
                  @Override
                  public void onResponse(String s) {
                      Log.d(TAG, s);
                  }
              },
              new Response.ErrorListener() {
                  @Override
                  public void onErrorResponse(VolleyError volleyError) {
                      Log.d(TAG, volleyError.getMessage(), volleyError);
                  }
              });
    mQueue.add(request);
```
## 分析
- Volley.newRequestQueue()方法
    有几个重载函数,最终是调用的这里,可以看到首先根据 sdk 版本是否大于等于9,去生成不同的 Network 对象.
    (可以看到在 sdk9 之前,不建议使用 HttpUrlConnection)
    然后初始化缓存目录,实例化 RequestQueue 对象,最后调用 start() 方法并 return queue.
    
    ```
        public static RequestQueue newRequestQueue(Context context, BaseHttpStack stack) {
            BasicNetwork network;
            if (stack == null) {
                if (Build.VERSION.SDK_INT >= 9) {
                    network = new BasicNetwork(new HurlStack());
                } else {
                    // Prior to Gingerbread, HttpUrlConnection was unreliable.
                    // See: http://android-developers.blogspot.com/2011/09/androids-http-clients.html
                    // At some point in the future we'll move our minSdkVersion past Froyo and can
                    // delete this fallback (along with all Apache HTTP code).
                    String userAgent = "volley/0";
                    try {
                        String packageName = context.getPackageName();
                        PackageInfo info = context.getPackageManager().getPackageInfo(packageName, 0);
                        userAgent = packageName + "/" + info.versionCode;
                    } catch (NameNotFoundException e) {
                    }
    
                    network = new BasicNetwork(
                            new HttpClientStack(AndroidHttpClient.newInstance(userAgent)));
                }
            } else {
                network = new BasicNetwork(stack);
            }
            return newRequestQueue(context, network);
        }
        private static RequestQueue newRequestQueue(Context context, Network network) {
                File cacheDir = new File(context.getCacheDir(), DEFAULT_CACHE_DIR);
                RequestQueue queue = new RequestQueue(new DiskBasedCache(cacheDir), network);
                queue.start();
                return queue;
        }
    ```
- Network/BasicNetwork/HurlStack/HttpClientStack 分别是啥?
    BasicNetwork 是 Network 的实现类,只有一个 performRequest 祖传方法,具体是根据 request 执行网络请求返回 NetworkResponse.
    ```
    public interface Network {
        /**
         * Performs the specified request.
         */
        NetworkResponse performRequest(Request<?> request) throws VolleyError;
    }
    ```
    HurlStack 和 HttpClientStack 是 HttpStack 的两个子类,分别是 HttpUrlConnection 的实现和 HttpClient的实现,
    也就是真正执行网络请求的地方,可以看出 Network 内部执行网络请求是调用了它.
    ```
    public interface HttpStack {
        /**
         * Performs an HTTP request with the given parameters.
         * @return the HTTP response
         */
        HttpResponse performRequest(Request<?> request, Map<String, String> additionalHeaders)
            throws IOException, AuthFailureError;
    
    }
    ```
- DiskBasedCache 是什么
    看名字可以猜出是缓存相关的,它是 Cache 的具体实现类,主要是管理缓存的,包括读取缓存,写入缓存,清除缓存等.
    ```
    public interface Cache {
        Entry get(String key);
        void put(String key, Entry entry);
        void initialize();
        void invalidate(String key, boolean fullExpire);
        void remove(String key);
        void clear();
        class Entry {
            //...
        }
    }
    ```
- RequestQueue 作用是什么
    分析到这里,就可以知道它是很重要的类了,它的构造函数需要传入 Cache 缓存管理对象, Network
    网络请求类, threadPoolSize 线程池大小, 和 ResponseDelivery 对象,
    ResponseDelivery 是第一次出现,它的作用就是将请求结果,response/error 返回给调用者.
    ```
    public RequestQueue(Cache cache, Network network, int threadPoolSize,
            ResponseDelivery delivery) {
        mCache = cache;
        mNetwork = network;
        mDispatchers = new NetworkDispatcher[threadPoolSize];
        mDelivery = delivery;
    }
    ```
    ```
    public interface ResponseDelivery {
        void postResponse(Request<?> request, Response<?> response);
        void postResponse(Request<?> request, Response<?> response, Runnable runnable);
        void postError(Request<?> request, VolleyError error);
    }
    ```
- RequestQueue.start() 方法做了什么
    start() 首先会调用 stop(), 然后实例化 CacheDispatcher 对象和 mDispatchers 数组,
    并分别调用它们的 start() 方法.
    ```
    private final NetworkDispatcher[] mDispatchers;
    private CacheDispatcher mCacheDispatcher;
    
    public void start() {
        stop();  // Make sure any currently running dispatchers are stopped.
        // Create the cache dispatcher and start it.
        mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue, mCache, mDelivery);
        mCacheDispatcher.start();
        // Create network dispatchers (and corresponding threads) up to the pool size.
        for (int i = 0; i < mDispatchers.length; i++) {
            NetworkDispatcher networkDispatcher = new NetworkDispatcher(mNetworkQueue, mNetwork,
                    mCache, mDelivery);
            mDispatchers[i] = networkDispatcher;
            networkDispatcher.start();
        }
    }
    public void stop() {
        if (mCacheDispatcher != null) {
            mCacheDispatcher.quit();
        }
        for (final NetworkDispatcher mDispatcher : mDispatchers) {
            if (mDispatcher != null) {
                mDispatcher.quit();
            }
        }
    }
    ```
- RequestQueue.add() 方法是直接执行请求了吗
    首先将 request 添加到 mCurrentRequests 集合,通过 setSequence()方法设置一个序列号,
    最后根据 shouldCache() 方法判断 request 是否需要缓存,将它添加到 mNetworkQueue 或 mCacheQueue 中.
    可以看到 add() 方法内没有执行网络请求的代码,大胆猜测下,一定是在 CacheDispatcher 和 NetworkDispatcher 类中
    ```
    private final Set<Request<?>> mCurrentRequests = new HashSet<Request<?>>();

    public <T> Request<T> add(Request<T> request) {
        // Tag the request as belonging to this queue and add it to the set of current requests.
        request.setRequestQueue(this);
        synchronized (mCurrentRequests) {
            mCurrentRequests.add(request);
        }
        // Process requests in the order they are added.
        request.setSequence(getSequenceNumber());
        request.addMarker("add-to-queue");

        // If the request is uncacheable, skip the cache queue and go straight to the network.
        if (!request.shouldCache()) {
            mNetworkQueue.add(request);
            return request;
        }
        mCacheQueue.add(request);
        return request;
     }
    ```
- CacheDispatcher 对象是保存缓存 request 吗
    