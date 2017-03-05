## 简单使用
Retrofit turns your HTTP API into a Java interface.
```Java
public interface GitHubService {
  @GET("users/{user}/repos")
  Call<List<Repo>> listRepos(@Path("user") String user);
}
```
The Retrofit class generates an implementation of the GitHubService interface.
```Java
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .addConverterFactory(GsonConverterFactory.create())
    .build();

GitHubService service = retrofit.create(GitHubService.class);
```
Each Call from the created GitHubService can make a synchronous or asynchronous HTTP request to the remote webserver.
```Java
Call<List<Repo>> call = service.listRepos("octocat");
List<Repo> repos = call.execute().body();
```

## 创建Retrofit对象
可以看到这里是通过一个builder模式来构建的Retrofit对象。  
主要就是传入和初始化各类参数。   
```Java
public Retrofit build() {
    if (baseUrl == null) {
        throw new IllegalStateException("Base URL required.");
    }

    // 创建OkHttpClient
    okhttp3.Call.Factory callFactory = this.callFactory;
    if (callFactory == null) {
        callFactory = new OkHttpClient();
    }

    Executor callbackExecutor = this.callbackExecutor;
    if (callbackExecutor == null) {
        // 这里可以理解成就是Handler
        callbackExecutor = platform.defaultCallbackExecutor();
    }

    // Make a defensive copy of the adapters and add the default Call adapter.
    List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
    adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

    // Make a defensive copy of the converters.
    List<Converter.Factory> converterFactories = new ArrayList<>(this.converterFactories);

    // 返回retrofit对象
    return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
      callbackExecutor, validateEagerly);
    }
}
```

## 获取API实例
这里在以前的一篇文章中分析过，主要是运用了动态代理的技术。

简而言之，就是动态生成接口的实现类（当然生成实现类有缓存机制），并创建其实例（称之为代理），代理把对接口的调用转发给 InvocationHandler 实例，而在 InvocationHandler 的实现中，除了执行真正的逻辑（例如再次转发给真正的实现类对象），我们还可以进行一些有用的操作，例如统计执行时间、进行初始化和清理、对接口调用进行检查等。

为什么要用动态代理？因为对接口的所有方法的调用都会集中转发到 InvocationHandler#invoke 函数中，我们可以集中进行处理，更方便了。你可能会想，我也可以手写这样的代理类，把所有接口的调用都转发到 InvocationHandler#invoke 呀，当然可以，但是可靠地自动生成岂不更方便？

[retrofit拆解 - 动态代理](http://www.jianshu.com/p/dcdd286d2e61#)   
[公共技术点之 Java 动态代理](http://a.codekk.com/detail/Android/Caij/%E5%85%AC%E5%85%B1%E6%8A%80%E6%9C%AF%E7%82%B9%E4%B9%8B%20Java%20%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86)

## 方法调用
在我们获取到API对象后，就可以直接调用方法进行网络请求了。所以我们就来看看这个网络请求到底是怎么发出去的。
调用接口中的方法的时候，就会进入到动态代理里面来
```Java
return (T) Proxy.newProxyInstance(service.getClassLoader(),
    new Class<?>[] { service },
    new InvocationHandler() {
      private final Platform platform = Platform.get();

      @Override
      public Object invoke(Object proxy, Method method, Object... args)
          throws Throwable {
        // If the method is a method from Object then defer to normal invocation.
        // 如果是object的方法，就直接调用
        if (method.getDeclaringClass() == Object.class) {
          return method.invoke(this, args);
        }
        // 如果是default方法，就直接调用default方法
        if (platform.isDefaultMethod(method)) {
          return platform.invokeDefaultMethod(method, service, proxy, args);
        }
        // 这里才是重点
        ServiceMethod serviceMethod = loadServiceMethod(method);
        OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
        return serviceMethod.callAdapter.adapt(okHttpCall);
      }
    });
```

### ServiceMethod
这里我们先了解一下ServiceMethod是有什么作用的：
>Adapts an invocation of an interface method into an HTTP call.  把对接口方法的调用转为一次 HTTP 调用。

我们可以理解成一个ServiceMethod对象对应于一个API interface的一个方法。  
至于这个ServiceMethod是怎么来的，就需要看看具体的方法了：
```Java
// 处理Retrofit.java中
ServiceMethod loadServiceMethod(Method method) {
    ServiceMethod result;
    synchronized (serviceMethodCache) {
        // 这里是有一个cache的。同一个API的同一个方法就只用解析一次
        result = serviceMethodCache.get(method);
        if (result == null) {
            // 这里也是通过builder模式得到ServiceMethod对象
          result = new ServiceMethod.Builder(this, method).build();
          serviceMethodCache.put(method, result);
        }
    }
    return result;
}
```
我们看看ServiceMethod的构造函数:
```Java
// 处于ServiceMethod.java
ServiceMethod(Builder<T> builder) {
    // 这里就是retrofit的callFactory, 其实就是okhttpClient
    this.callFactory = builder.retrofit.callFactory();
    // 这个就联想RxJavaCallAdapterFactory，它是把
    this.callAdapter = builder.callAdapter;
    this.baseUrl = builder.retrofit.baseUrl();
    // 负责把服务器返回的数据（JSON、XML、二进制或者其他格式，由 ResponseBody 封装）转化为 T 类型的对象
    this.responseConverter = builder.responseConverter;
    this.httpMethod = builder.httpMethod;
    this.relativeUrl = builder.relativeUrl;
    this.headers = builder.headers;
    this.contentType = builder.contentType;
    this.hasBody = builder.hasBody;
    this.isFormEncoded = builder.isFormEncoded;
    this.isMultipart = builder.isMultipart;
    // 则负责解析 API 定义时每个方法的参数，并在构造 HTTP 请求时设置参数；
    this.parameterHandlers = builder.parameterHandlers;
}
```
#### callFactory
callFactory 实际上由 Retrofit 类提供，而我们在构造 Retrofit 对象时，可以指定 callFactory，如果不指定，将默认设置为一个 okhttp3.OkHttpClient。

#### callAdapter
```java
// 处于ServiceMethod.java
private CallAdapter<?> createCallAdapter() {
  // 省略检查性代码
  Annotation[] annotations = method.getAnnotations();
  try {
    return retrofit.callAdapter(returnType, annotations);
  } catch (RuntimeException e) {
    // Wide exception range because factories are user code.
    throw methodError(e, "Unable to create call adapter for %s", returnType);
  }
}

// 处理Retrofit.java
public CallAdapter<?, ?> callAdapter(Type returnType, Annotation[] annotations) {
    return nextCallAdapter(null, returnType, annotations);
}

public CallAdapter<?, ?> nextCallAdapter(CallAdapter.Factory skipPast, Type returnType,
      Annotation[] annotations) {
    checkNotNull(returnType, "returnType == null");
    checkNotNull(annotations, "annotations == null");

    int start = adapterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = adapterFactories.size(); i < count; i++) {
        // 这里进入到具体的CallAdapter中去获取类型（java，rxjava，guava）
        CallAdapter<?, ?> adapter = adapterFactories.get(i).get(returnType, annotations, this);
        if (adapter != null) {
            return adapter;
        }
    }
    // 抛出异常
    ...
    throw new IllegalArgumentException(builder.toString());
}
```

#### responseConverter
```java
private Converter<ResponseBody, T> createResponseConverter() {
  Annotation[] annotations = method.getAnnotations();
  try {
    return retrofit.responseBodyConverter(responseType, annotations);
  } catch (RuntimeException e) {
    // Wide exception range because factories are user code.
    throw methodError(e, "Unable to create converter for %s", responseType);
  }
}
```
同样，responseConverter 还是由 Retrofit 类提供，而在其内部，逻辑和创建 callAdapter 基本一致，通过遍历 Converter.Factory 列表，看看有没有工厂能够提供需要的 responseBodyConverter。工厂列表同样可以在构造 Retrofit 对象时进行添加。

#### parameterHandlers
每个参数都会有一个 ParameterHandler，由 ServiceMethod#parseParameter 方法负责创建，其主要内容就是解析每个参数使用的注解类型（诸如 Path，Query，Field 等），对每种类型进行单独的处理。构造 HTTP 请求时，我们传递的参数都是字符串，那 Retrofit 是如何把我们传递的各种参数都转化为 String 的呢？还是由 Retrofit 类提供 converter！

Converter.Factory 除了提供上一小节提到的 responseBodyConverter，还提供 requestBodyConverter 和 stringConverter，API 方法中除了 @Body 和 @Part 类型的参数，都利用 stringConverter 进行转换，而 @Body 和 @Part 类型的参数则利用 requestBodyConverter 进行转换。

### OkHttpCall
```java
final class OkHttpCall<T> implements Call<T> {
    OkHttpCall(ServiceMethod<T, ?> serviceMethod, Object[] args) {
        this.serviceMethod = serviceMethod;
        this.args = args;
    }
}
```
OkHttpCall 实现了 retrofit2.Call，我们通常会使用它的 execute() 和 enqueue(Callback<T> callback) 接口。前者用于同步执行 HTTP 请求，后者用于异步执行。  


### serviceMethod.callAdapter.adapt
CallAdapter<T>#adapt(Call<R> call) 函数负责把 retrofit2.Call<R> 转为 T。这里 T 当然可以就是 retrofit2.Call<R>，这时我们直接返回参数就可以了，实际上这正是 DefaultCallAdapterFactory 创建的 CallAdapter 的行为。  

其实就是返回接口中定义的方法的返回值，要么是Call<T>，要么是Observable<T>

这里我们就以默认的`ExecutorCallAdapterFactory  `来进行分析：
```java
// 这里是在ExecutorCallAdapterFactory.java
public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (getRawType(returnType) != Call.class) {
      return null;
    }
    final Type responseType = Utils.getCallResponseType(returnType);
    return new CallAdapter<Object, Call<?>>() {
      @Override public Type responseType() {
        return responseType;
      }

      // 在这里进行的adapt方法
      // 参数call其实就是OkHttpCall
      @Override public Call<Object> adapt(Call<Object> call) {
          // 其实这里返回的就是一个CallbackCall
        return new ExecutorCallbackCall<>(callbackExecutor, call);
      }
    };
}
```

### enqueue
```java
// 这里是在ExecutorCallAdapterFactory.java
static final class ExecutorCallbackCall<T> implements Call<T> {
    final Executor callbackExecutor;
    final Call<T> delegate;

    ExecutorCallbackCall(Executor callbackExecutor, Call<T> delegate) {
      this.callbackExecutor = callbackExecutor;
      this.delegate = delegate;
    }

    @Override public void enqueue(final Callback<T> callback) {
      if (callback == null) throw new NullPointerException("callback == null");
      // 这个delegate 其实就是OkHttpCall, OkHttpCall就是对okhttp的包装
      delegate.enqueue(new Callback<T>() {
        @Override public void onResponse(Call<T> call, final Response<T> response) {
            // 这里的callbackExecutor 其实就是线程调度器，在Android平台下默认就是Handler
            // execute其实就是handler.post(r)
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              if (delegate.isCanceled()) {
                // Emulate OkHttp's behavior of throwing/delivering an IOException on cancellation.
                callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
              } else {
                callback.onResponse(ExecutorCallbackCall.this, response);
              }
            }
          });
        }

        @Override public void onFailure(Call<T> call, final Throwable t) {
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              callback.onFailure(ExecutorCallbackCall.this, t);
            }
          });
        }
      });
    }
```
这里其实就是对应了用户调用的enqueue方法，  
内部先是通过OkHttpCall调用enqueue，
然后OkHttpCall内部还是由okhttp的enqueue方法
```java
// 这里是在OkHttpCall中执行的
@Override public void enqueue(final Callback<T> callback) {
    if (callback == null) throw new NullPointerException("callback == null");

    okhttp3.Call call;
    Throwable failure;

    synchronized (this) {
      if (executed) throw new IllegalStateException("Already executed.");
      executed = true;

      call = rawCall;
      failure = creationFailure;
      // 创建真正的OKHttp
      if (call == null && failure == null) {
        try {
            // 在这里用convert对请求进行了封装
          call = rawCall = createRawCall();
        } catch (Throwable t) {
          failure = creationFailure = t;
        }
      }
    }

    if (failure != null) {
      callback.onFailure(this, failure);
      return;
    }

    if (canceled) {
      call.cancel();
    }
    // okhttp执行真正的enqueue方法
    call.enqueue(new okhttp3.Callback() {
      @Override public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse)
          throws IOException {
        Response<T> response;
        try {
            // 解析返回数据
          response = parseResponse(rawResponse);
        } catch (Throwable e) {
          callFailure(e);
          return;
        }
        callSuccess(response);
      }

      @Override public void onFailure(okhttp3.Call call, IOException e) {
        try {
          callback.onFailure(OkHttpCall.this, e);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }

      private void callFailure(Throwable e) {
        try {
          callback.onFailure(OkHttpCall.this, e);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }

      private void callSuccess(Response<T> response) {
        try {
          callback.onResponse(OkHttpCall.this, response);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }
    });
  }
```

### Converter
retrofit 模块内置了 BuiltInConverters，只能处理 ResponseBody， RequestBody 和 String 类型的转化（实际上不需要转）。而 retrofit-converters 中的子模块则提供了 JSON，XML，ProtoBuf 等类型数据的转换功能，而且还有多种转换方式可以选择。这里我主要关注 GsonConverterFactory。

根据上面的enqueue里的代码，里面会有一个解析数据的方法，我们先看看那个方法：
```java
// 这里是在OkHttpCall.java中
Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    ResponseBody rawBody = rawResponse.body();

    // Remove the body's source (the only stateful object) so we can pass the response along.
    rawResponse = rawResponse.newBuilder()
        .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
        .build();

    int code = rawResponse.code();
    if (code < 200 || code >= 300) {
      try {
        // Buffer the entire body to avoid future I/O.
        ResponseBody bufferedBody = Utils.buffer(rawBody);
        return Response.error(bufferedBody, rawResponse);
      } finally {
        rawBody.close();
      }
    }

    if (code == 204 || code == 205) {
      rawBody.close();
      return Response.success(null, rawResponse);
    }

    ExceptionCatchingRequestBody catchingBody = new ExceptionCatchingRequestBody(rawBody);
    try {
        // 重点关注这个!!!
      T body = serviceMethod.toResponse(catchingBody);
      return Response.success(body, rawResponse);
    } catch (RuntimeException e) {
      // If the underlying source threw an exception, propagate that rather than indicating it was
      // a runtime exception.
      catchingBody.throwIfCaught();
      throw e;
    }
}

// 这里是在ServiceMethod.java
/** Builds a method return value from an HTTP response body. */
R toResponse(ResponseBody body) throws IOException {
    // 所以这里就可以直接关注convert.convert方法了
    return responseConverter.convert(body);
}
```

## 小结
![整体流程图](http://upload-images.jianshu.io/upload_images/625299-29a632638d9f518f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 相关链接

[拆轮子系列：拆 Retrofit](https://blog.piasy.com/2016/06/25/Understand-Retrofit/)

[Retrofit分析-漂亮的解耦套路](http://www.jianshu.com/p/45cb536be2f4#)
