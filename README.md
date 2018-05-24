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
    并分别启动线程,默认 mDispatchers 长度是4,也就是启动一个缓存线程,4个工作线程.
    注意 mCacheQueue 和 mNetworkQueue 都是优先阻塞队列,且4个工作线程公用 mNetworkQueue 
    ```
    private final NetworkDispatcher[] mDispatchers;
    private CacheDispatcher mCacheDispatcher;
    private final PriorityBlockingQueue<Request<?>> mCacheQueue =
                new PriorityBlockingQueue<>();
    private final PriorityBlockingQueue<Request<?>> mNetworkQueue =
            new PriorityBlockingQueue<>();

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
- RequestQueue.add() 方法做了些什么

    首先将 request 添加到 mCurrentRequests 集合,通过 setSequence()方法设置一个序列号,
    最后根据 shouldCache() 方法判断 request 是否需要缓存,如果需要将它添加到 mCacheQueue 中,否则添加到 mNetworkQueue 中.
    然后就完了,返回 request 对象.
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
- 添加 request 后，NetworkDispatcher 和 CacheDispatcher 是怎么处理的？
    在上一步中，request 根据缓存添加到 mCacheQueue 或 mNetworkQueue 阻塞队列中，这两个阻塞队列是被关联在 CacheDispatcher 和 NetworkDispatcher 中了。
    可以想到这两分发器肯定是阻塞取出请求做处理。
    - CacheDispatcher
        主要看它的 run() 方法
        ```java
        public void run() {
              // 省略部分代码
            while (true) {
              final Request<?> request = mCacheQueue.take();   
              // 省略部分代码
              if (request.isCanceled()) {
                  request.finish("cache-discard-canceled");
                  continue;
              }
              Cache.Entry entry = mCache.get(request.getCacheKey());
              if (entry == null) {
                request.addMarker("cache-miss");
                // Cache miss; send off to the network dispatcher.
                if (!mWaitingRequestManager.maybeAddToWaitingRequests(request)) {
                    mNetworkQueue.put(request);
                }
                continue;
              }
              .....
              mDelivery.postResponse(request, response);
            }
      }
        ```
        和我猜想的一样，死循环，从阻塞队列中取出 request ，然后再做处理，判断是否去掉，判断是否过期，判断缓存中是否存在，如果不存在则加入到 mNetworkQueue ，
        代表需要联网获取，否则，直接回调 mDelivery.postResponse 也就是结束的回调。
        
        总结一下,CacheDispatcher 主要是判断缓存是否可用,如果可用直接返回,不用网络请求了,否则,
        会将 request 丢给 mNetworkQueue, 最终是由 NetworkDispatcher 处理的.
    - NetworkDispatcher
    它的 run 方法和上述 CacheDispatcher 类似，只不过是请求网络，再判断是否需要缓存，如果需要则保存，最后回调 mDelivery.
    
- ExecutorDelivery 是怎么处理响应的
    ExecutorDelivery 的默认实现是 NetworkDispatcher,在 RequestQueue 构造方法里可以看到,
    传入了主线程的 Handler 对象,在 postResponse 和 postError 方法中,实际上调用了 handler.post()方法,
    在主线程执行,而 ResponseDeliveryRunnable 的 run 方法里,
    通过判断 mResponse.isSuccess() 请求是否成功,回调 request 的 deliverResponse() 
    和 deliverError()方法,最终回调了我们自定义的回调方法.
    
    总结一下,它的作用就是:将响应的结果 response 回调给 request 中的回调方法,并在主线程中进行. 
    
    ```
    private final Executor mResponsePoster;
    mResponsePoster = new Executor() {
            @Override
            public void execute(Runnable command) {
                handler.post(command);
            }
        };
    @Override
    public void postResponse(Request<?> request, Response<?> response, Runnable runnable) {
        request.markDelivered();
        request.addMarker("post-response");
        mResponsePoster.execute(new ResponseDeliveryRunnable(request, response, runnable));
    }

    @Override
    public void postError(Request<?> request, VolleyError error) {
        request.addMarker("post-error");
        Response<?> response = Response.error(error);
        mResponsePoster.execute(new ResponseDeliveryRunnable(request, response, null));
    }
    private class ResponseDeliveryRunnable implements Runnable {
        private final Request mRequest;
        private final Response mResponse;
        private final Runnable mRunnable;

        public ResponseDeliveryRunnable(Request request, Response response, Runnable runnable) {
            mRequest = request;
            mResponse = response;
            mRunnable = runnable;
        }
        @Override
        public void run() {
            if (mRequest.isCanceled()) {
                mRequest.finish("canceled-at-delivery");
                return;
            }
            if (mResponse.isSuccess()) {
                mRequest.deliverResponse(mResponse.result);
            } else {
                mRequest.deliverError(mResponse.error);
            }
            // ignore some code
       }
    }
    ```
    
- 并发时怎么处理的?
    由于采用了阻塞队列，所有的联网请求最终都是添加到 mNetworkQueue 这个队列中的，默认工作分发器 NetworkDispatcher 有4个，它们会从这个队列中取出请求，然后处理。
    打个比方，当有一堆工作来的时候，4个人同时处理，且保证独立，也就是一个工作不会2个人同时做。
    
- HTTP 响应内容的解析在哪里？
    在 request 的 parseNetworkResponse 里，比如 StringRequest 就是最简单的。这个方法在 NetworkDispatcher run 处理请求完后调用，并最终传递给 ResponseDelivery ，再到回调
    