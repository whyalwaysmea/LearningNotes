# OKHttp简单使用
```java
//创建OkHttpClient
OkHttpClient client = new OkHttpClient.Builder().build();

//创建Request
Request request = new Request.Builder()
        .url("http://publicobject.com/helloworld.txt")
        .build();

//创建Call
Call call = client.newCall(request);

//执行Call的异步方法
 call.enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                //做一些请求失败的处理
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
              //做一些请求成功的处理
            }
        });
```

OkHttpClient和Request其实比较简单，常见的Build模式，配置相关参数就完事。我们直接开始研究Call吧。

# Call#RealCall
我们进入OKHttpClient.java可以发现以下这么一段代码:
```java
/**
* Prepares the {@code request} to be executed at some point in the future.
*/
@Override public Call newCall(Request request) {
    return new RealCall(this, request);
}
```
毫无疑问，我们所调用的`client.newCall(request)`实际就是上面那个方法，可以看到，它其实是调用的`new RealCall(this, request);`

那么自然我们就应该去看看RealCall这是个什么东西了，不过，我们可以先来看看Call是什么吧。因为RealCall肯定跟Call是有某种关系的
```java
/**
 * A call is a request that has been prepared for execution. A call can be canceled. As this object
 * represents a single request/response pair (stream), it cannot be executed twice.
 */
public interface Call extends Cloneable {
  Request request();

  Response execute() throws IOException;

  void enqueue(Callback responseCallback);

  void cancel();

  boolean isExecuted();

  boolean isCanceled();

  Call clone();

  interface Factory {
    Call newCall(Request request);
  }
}
```
Call是一个接口，基本上我们会用到的大部分操作都定义在这个里面了，可以说这个接口是OkHttp框架的操作核心。在构造完HttpClient和Request之后，我们只要持有Call对象的引用就可以操控请求了。
自然而然的就会明白，RealCall就是Call的实现类了。我们可以看看其中的一些比较重要的方法:
```java
final class RealCall implements Call {
    // 可见，call是持有OKHttpClient和Request的
    protected RealCall(OkHttpClient client, Request originalRequest) {
        this.client = client;
        this.originalRequest = originalRequest;
    }
    @Override
    public Response execute() throws IOException {
        synchronized (this) {
            // 检查这个 call 是否已经被执行了，每个 call 只能被执行一次
            if (executed) throw new IllegalStateException("Already Executed");
            executed = true;
        }
        try {
            // 利用 client.dispatcher().executed(this) 来进行实际执行，dispatcher 是刚才看到的 OkHttpClient.Builder 的成员之一
            client.dispatcher().executed(this);
            // 调用 getResponseWithInterceptorChain() 函数获取 HTTP 返回结果，从函数名可以看出，这一步还会进行一系列“拦截”操作。
            Response result = getResponseWithInterceptorChain(false);
            if (result == null) throw new IOException("Canceled");
            return result;
        } finally {
            // 最后还要通知 dispatcher 自己已经执行完毕。
            client.dispatcher().finished(this);
        }
    }

    @Override
    public void enqueue(Callback responseCallback) {
        enqueue(responseCallback, false);
    }

    void enqueue(Callback responseCallback, boolean forWebSocket) {
        synchronized (this) {
            if (executed) throw new IllegalStateException("Already Executed");
            executed = true;
        }
        // 这里我们需要了解Dispatcher和AsyncCall
        client.dispatcher().enqueue(new AsyncCall(responseCallback, forWebSocket));
    }
}
```
通过以上代码，其实*execute*方法是很好理解的，因为他只涉及到了`Dispatcher`和`getResponseWithInterceptorChain(false);`
*enqueue* 看来有点麻烦，因为除了`Dispatcher`以外，还有一个`AsyncCall`。所以我们就针对这几个东西再来进一步了解

# Dispatcher
OkHttp的dispatcher参数是直接new出来的。
```java
/**
 * Policy on when async requests are executed.
 *
 * <p>Each dispatcher uses an {@link ExecutorService} to run calls internally. If you supply your
 * own executor, it should be able to run {@linkplain #getMaxRequests the configured maximum} number
 * of calls concurrently.
 */
public final class Dispatcher {
  /** 最大并发请求数为64 */
  private int maxRequests = 64;
  /** 每个主机最大请求数为5 */
  private int maxRequestsPerHost = 5;

  /** 线程池 */
  private ExecutorService executorService;

  /** 准备执行的请求 */
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  /** 正在执行的异步请求，包含已经取消但未执行完的请求 */
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  /** 正在执行的同步请求，包含已经取消单未执行完的请求 */
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
  synchronized void enqueue(AsyncCall call) {
    //如果正在执行的请求数小于设定值，
    //并且请求同一个主机的request小于设定值
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
        //添加到执行队列，开始执行请求
        runningAsyncCalls.add(call);
        // 执行线程
        // 调用ExecutorService的execute(call)方法实际上最后调用的就是AsyncCall 的execute()方法。
        executorService().execute(call);
    } else {
        // 添加到等待队列中
        readyAsyncCalls.add(call);
    }
  }

  public synchronized ExecutorService executorService() {
    if (executorService == null) {
        // 构造一个线程池
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
      new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }
}
```
在分发器中，它会根据情况决定把call加入请求队列还是等待队列，在请求队列中的话，就会在线程池中执行这个请求。

# RealCall#AsyncCall
```java
//NamedRunnable实现了Runnable接口，把run()方法封装成了execute()
final class AsyncCall extends NamedRunnable {
    private final Callback responseCallback;
    private final boolean forWebSocket;

    private AsyncCall(Callback responseCallback, boolean forWebSocket) {
      super("OkHttp %s", redactedUrl().toString());
      this.responseCallback = responseCallback;
      this.forWebSocket = forWebSocket;
    }
    ...

    @Override
    protected void execute() {
        boolean signalledCallback = false;
        try {
          // 返回response
          Response response = getResponseWithInterceptorChain(forWebSocket);
          if (canceled) {
            signalledCallback = true;
             responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
          } else {
            signalledCallback = true;
            responseCallback.onResponse(RealCall.this, response);
          }
        } catch (IOException e) {
          if (signalledCallback) {
            // Do not signal the callback twice!
            Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
          } else {
            responseCallback.onFailure(RealCall.this, e);
          }
        } finally {
          client.dispatcher().finished(this);
        }
    }
}
```
## getResponseWithInterceptorChain / Interceptor
```java
private Response getResponseWithInterceptorChain(boolean forWebSocket) throws IOException {
    Interceptor.Chain chain = new ApplicationInterceptorChain(0, originalRequest, forWebSocket);
    return chain.proceed(originalRequest);
}
```
从方法名字基本可以猜到是干嘛的，调用chain.proceed(originalRequest);将request传递进来，从拦截器链里拿到返回结果。那么拦截器Interceptor是干嘛的，Chain是干嘛的呢？继续往下看ApplicationInterceptorChain
```java
class ApplicationInterceptorChain implements Interceptor.Chain {
    private final int index;
    private final Request request;

    ApplicationInterceptorChain(int index, Request request, boolean forWebSocket) {
      this.index = index;
      this.request = request;
      this.forWebSocket = forWebSocket;
    }

    @Override public Connection connection() {
      return null;
    }

    @Override public Request request() {
      return request;
    }

    @Override public Response proceed(Request request) throws IOException {
      // If there's another interceptor in the chain, call that.
      if (index < client.interceptors().size()) {
        Interceptor.Chain chain = new ApplicationInterceptorChain(index + 1, request, forWebSocket);
        Interceptor interceptor = client.interceptors().get(index);
        Response interceptedResponse = interceptor.intercept(chain);

        if (interceptedResponse == null) {
          throw new NullPointerException("application interceptor " + interceptor
              + " returned null");
        }

        return interceptedResponse;
      }

      // No more interceptors. Do HTTP.
      return getResponse(request, forWebSocket);
    }
  }
```
ApplicationInterceptorChain实现了Interceptor.Chain接口，持有Request的引用。
```java
public interface Interceptor {
  Response intercept(Chain chain) throws IOException;

  interface Chain {
    Request request();

    Response proceed(Request request) throws IOException;

    Connection connection();
  }
}
```
Interceptor是拦截者的意思，就是把Request请求或者Response回复做一些处理，而OkHttp通过一个“链条”Chain把所有的Interceptor串联在一起，保证所有的Interceptor一个接着一个执行。

这个设计突然之间就把问题分解了，在这种机制下，所有繁杂的事物都可以归类，每个Interceptor只执行一小类事物。这样，每个Interceptor只关注自己份内的事物，问题的复杂度一下子降低了几倍。而且这种插拔的设计，极大的提高了程序的可拓展性。

proceed方法中判断index（此时为0）是否小于client.interceptors(List )的大小，如果小于也就是说client.interceptors还有Interceptor，那么就再封装一个ApplicationInterceptorChain，只不过index + 1，然后取出第index个Interceptor将chain传递进去。传递进去干嘛呢？我们看一个用法，以实际项目为例
```java
HttpLoggingInterceptor interceptor = new HttpLoggingInterceptor(new RetrofitLogger());
        interceptor.setLevel(HttpLoggingInterceptor.Level.BODY);
OkHttpClient client = new OkHttpClient.Builder()
        .addInterceptor(interceptor)
        .retryOnConnectionFailure(true)
        .connectTimeout(15, TimeUnit.SECONDS)
        .addInterceptor(getCommonParameterInterceptor())
        .addNetworkInterceptor(getTokenInterceptor())
        .build();

@Override
protected Interceptor getCommonParameterInterceptor() {
    return new Interceptor() {
        @Override
        public Response intercept(Chain chain) throws IOException {
            Request originalRequest = chain.request();
            Request request = originalRequest;
            if (!originalRequest.method().equalsIgnoreCase("POST")) {
                HttpUrl modifiedUrl = originalRequest.url().newBuilder()
                        .addQueryParameter("version_code", String.valueOf(AppUtils.getVersionCode()))
                        .addQueryParameter("app_key", "nicepro")
                        .addQueryParameter("app_device", "Android")
                        .addQueryParameter("app_version", AppUtils.getVersionName())
                        .addQueryParameter("token", AccountUtils.getToken())
                        .build();
                request = originalRequest.newBuilder().url(modifiedUrl).build();
            }
            return chain.proceed(request);
        }
    };
}

@Override
protected Interceptor getTokenInterceptor() {
    return new Interceptor() {
        @Override
        public Response intercept(Chain chain) throws IOException {
            Request originalRequest = chain.request();
            Request authorised = originalRequest.newBuilder()
                    .header("app-key", "nicepro")
                    .header("app-device", "Android")
                    .header("app-version", AppUtils.getVersionName())
                    .header("os", AppUtils.getOs())
                    .header("os-version", AppUtils.getAndroidVersion() + "")
                    .header("Accept", "application/json")
                    .header("User-Agent", "Android/retrofit")
                    .header("token", AccountUtils.getToken())
                    .build();
            return chain.proceed(authorised);
        }
    };
}
```
可以看到每个Interceptor的intercept方法中做了一些操作后，最后都会调用chain.proceed(request)方法，而这个chain就是每次prceed方法中生成的ApplicationInterceptorChain，用index+1的方式递归调用OkHttClient中的Interceptors，进行拦截操作，比如可以用来监控log，修改请求，修改结果，供开发者自定义参数添加等等，然后最终调用的还是最初的index=0的那个chain的proceed方法中的getResponse(request, forWebSocket);。

可以说OkHttp是用chain串联起拦截器，而每个拦截器都有能力返回Response，返回Response即终止整个调用链，这种设计模式称为责任链模式。这种模式为OkHttp提供了强大的装配能力，极大的提高了OkHttp的扩展性和可维护性。

在Android系统中最典型的责任链模式就是View的Touch传递机制，一层一层传递直到被消费。


# 相关链接
[OkHttp3 源码浅析](http://w4lle.github.io/2016/12/06/OkHttp/)

[拆轮子系列：拆 OkHttp](http://blog.piasy.com/2016/07/11/Understand-OkHttp/)

[从OKHttp框架看代码设计](https://gold.xitu.io/post/581311cabf22ec0068826aff)
