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
    return new RealCall(this, request, fasle /*for web socket*/);
}
```
毫无疑问，我们所调用的`client.newCall(request)`实际就是上面那个方法，可以看到，它其实是调用的`new RealCall(this, request, false);`

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
    RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
        this.client = client;
        this.originalRequest = originalRequest;
        // 下面这两句是3.4之后才加入的
        this.forWebSocket = forWebSocket;
        this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);
    }
    @Override
    public Response execute() throws IOException {
        synchronized (this) {
            // 检查这个 call 是否已经被执行了，每个 call 只能被执行一次
            if (executed) throw new IllegalStateException("Already Executed");
            executed = true;
        }
        captureCallStackTrace();
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
        synchronized (this) {
          if (executed) throw new IllegalStateException("Already Executed");
          executed = true;
        }
        captureCallStackTrace();
        client.dispatcher().enqueue(new AsyncCall(responseCallback));
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

    private AsyncCall(Callback responseCallback) {
      super("OkHttp %s", redactedUrl().toString());
      this.responseCallback = responseCallback;      
    }
    ...

    @Override
    protected void execute() {
        boolean signalledCallback = false;
        try {
          // 返回response
          Response response = getResponseWithInterceptorChain(forWebSocket);
          if (retryAndFollowUpInterceptor.isCanceled()) {
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
### 3.3版本如下
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
  //只有一个接口方法
  Response intercept(Chain chain) throws IOException;

  interface Chain {
    //  Chain其实包装了一个Request请求
    Request request();
    // 获取Response
    Response proceed(Request request) throws IOException;
    // 获得当前网络连接
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

### 3.4版本如下
```java
Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    // 添加开发者应用者自定义的Interceptor
    interceptors.addAll(client.interceptors());
    // 处理请求失败的重试，重定向的Interceptor
    interceptors.add(retryAndFollowUpInterceptor);
    // 添加一些请求的头部或者其他信息
    // 并对返回的Response做一些友好的处理
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    // 判断缓存是否存在，读取缓存，更新缓存等等
    interceptors.add(new CacheInterceptor(client.internalCache()));
    // 建立客户端和服务器的连接
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
        // 添加开发者自定义的网络层拦截
      interceptors.addAll(client.networkInterceptors());
    }
    // 向服务器发送数据，并且接收服务器返回的Response
    interceptors.add(new CallServerInterceptor(forWebSocket));
    // 一个包裹这request的chain
    Interceptor.Chain chain = new RealInterceptorChain(
        interceptors, null, null, null, 0, originalRequest);
    //把chain传递到第一个Interceptor手中    
    return chain.proceed(originalRequest);
  }
```
到这里，我们通过源码已经可以总结一些在开发中需要注意的问题了：
* Interceptor的执行的是顺序的，也就意味着当我们自己自定义Interceptor时是否应该注意添加的顺序呢？
* 在开发者自定义拦截器时，是有两种不同的拦截器可以自定义的。

接着，从上面最后两行代码讲起：

首先创建了一个指向RealInterceptorChain这个实现类的chain引用，然后调用了 proceed（request）方法。

```java
public final class RealInterceptorChain implements Interceptor.Chain {
  private final List<Interceptor> interceptors;
  private final StreamAllocation streamAllocation;
  private final HttpCodec httpCodec;
  private final Connection connection;
  private final int index;
  private final Request request;
  private int calls;

  public RealInterceptorChain(List<Interceptor> interceptors, StreamAllocation streamAllocation,
      HttpCodec httpCodec, Connection connection, int index, Request request) {
    this.interceptors = interceptors;
    this.connection = connection;
    this.streamAllocation = streamAllocation;
    this.httpCodec = httpCodec;
    this.index = index;
    this.request = request;
  }
  ....
  ....
  ....
 @Override
 public Response proceed(Request request) throws IOException {
            //直接调用了下面的proceed（.....）方法。
    return proceed(request, streamAllocation, httpCodec, connection);
  }

    //这个方法用来获取list中下一个Interceptor，并调用它的intercept（）方法
  public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      Connection connection) throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();

    calls++;
    ....
    ....
    ....

    // Call the next interceptor in the chain.
    RealInterceptorChain next = new RealInterceptorChain(
        interceptors, streamAllocation, httpCodec, connection, index + 1, request);
    //从list中获取到第一个Interceptor
    Interceptor interceptor = interceptors.get(index);
    //然后调用这个Interceptor的intercept（）方法，并等待返回Response
    Response response = interceptor.intercept(next);
    ....
    ....
    return response;
  }
```
如果你还是好奇OKHttp到底是怎么发出请求？
我可以做一点简短的介绍：这个请求动作发生在CallServerInterceptor（也就是最后一个Interceptor）中，而且其中还涉及到Okio这个io框架，通过Okio封装了流的读写操作，可以更加方便，快速的访问、存储和处理数据。最终请求调用到了socket这个层次,然后获得Response。

### 两个版本之间的区别
在或之前的源码中，从请求到响应会嵌套了很多方法，并且有两个 chain，一个传给 interceptor，一个传给 networkinterceptor，并且两个 interceptor 处理的位置都不一样，光 networkinterceptor 的调用位置我都找了半天，总之看代码真的需要耐心，现在只通过一个 chain 并且通过不断传给在不同顺序的 interceptor，每个 interceptor 做不同的操作来解决整个操作，逻辑清晰

# Q&A
### 每个 body 只能被消费一次，多次消费会抛出异常, [why?](https://github.com/square/okhttp/issues/1240#issuecomment-233655904)
```java
if (!forWebSocket || response.code() != 101) {
  response = response.newBuilder()
      .body(httpCodec.openResponseBody(response))
      .build();
}
```
由 HttpCodec#openResponseBody 提供具体 HTTP 协议版本的响应 body，而 HttpCodec 则是利用 Okio 实现具体的数据 IO 操作。

ResponseBody必须关闭，不然可能造成资源泄漏

如果ResponseBody中的数据很大，则不应该使用bytes() 或 string()方法，它们会将结果一次性读入内存，而应该使用byteStream()或 charStream()，以流的方式读取数据。

Because response body can be huge so OkHttp doesn’t store it in memory, it reads it as a stream from network when you need it.

When you read body as a string() OkHttp downloads response body and returns it to you without keeping reference to the string, it can’t be downloaded twice without new request.

### 每个Call对象只能执行一次请求 why?

# 相关链接
[OkHttp3 源码浅析](http://w4lle.github.io/2016/12/06/OkHttp/)

[拆轮子系列：拆 OkHttp](http://blog.piasy.com/2016/07/11/Understand-OkHttp/)

[从OKHttp框架看代码设计](https://gold.xitu.io/post/581311cabf22ec0068826aff)

[重识OkHttp——探究源码设计](http://www.jianshu.com/p/c58fd0a78791#)
