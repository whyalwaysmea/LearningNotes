为了更好的讲解OKHTTP怎么设置缓存，我们追根溯源先从浏览器的缓存说起，这样后面的OKHTTP缓存内容自然更加好理解。
*缓存分类*: http请求有服务端和客户端之分。因此缓存也可以分为两个类型服务端侧和客户端侧。

## 服务端缓存
常见的服务端有Ngix和Apache。服务端缓存又分为代理服务器缓存和反向代理服务器缓存。常见的CDN就是服务器缓存。这个好理解，当浏览器重复访问一张图片地址时，CDN会判断这个请求有没有缓存，如果有的话就直接返回这个缓存的请求回复，而不再需要让请求到达真正的服务地址，这么做的目的是减轻服务端的运算压力。

## 客户端缓存
客户端主要指浏览器（如IE、Chrome等），当然包括我们的OKHTTPClient.客户端第一次请求网络时，服务器返回回复信息。如果数据正常的话，客户端缓存在本地的缓存目录。当客户端再次访问同一个地址时，客户端会检测本地有没有缓存，如果有缓存的话，数据是有没有过期，如果没有过期的话则直接运用缓存内容。

## 缓存中的重要概念
#### 1.Expires
表示到期时间，一般用在response报文中，当超过此事件后响应将被认为是无效的而需要网络连接，反之而是直接使用缓存
#### 2.Cache-Control
相对值，单位是秒，指定某个文件被续多少秒的时间，从而避免额外的网络请求。比expired更好的选择，它不用要求服务器与客户端的时间同步，也不用服务器时刻同步修改配置Expired中的绝对时间，而且它的优先级比Expires更高。

Expires 的一个缺点就是，返回的到期时间是服务器端的时间，这样存在一个问题，如果客户端的时间与服务器的时间相差很大，那么误差就很大，所以在HTTP 1.1版开始，使用Cache-Control: max-age=秒替代。

**Cache-Control：** 中又有以下一些值：
* public ---- 数据内容皆被储存起来，就连有密码保护的网页也储存，安全性很低
* private ---- 数据内容只能被储存到私有的cache，仅对某个用户有效，不能共享
* no-cache ---- 可以缓存，但是只有在跟WEB服务器验证了其有效后，才能返回给客户端
* no-store ---- 请求和响应都禁止被缓存
* max-age： ----- 本响应包含的对象的过期时间
* max-stale ---- 允许读取过期时间必须小于max-stale 值的缓存对象。
* no-transform ---- 告知代理,不要更改媒体类型,比如jpg,被你改成png.

#### 3.Last-Modified/If-Modified-Since
这个需要配合Cache-Control使用
**Last-Modified**：标示这个响应资源的最后修改时间。web服务器在响应请求时，告诉浏览器资源的最后修改时间。

**If-Modified-Since**：当资源过期时（使用Cache-Control标识的max-age），发现资源具有Last-Modified声明，则再次向web服务器请求时带上头 If-Modified-Since，表示请求时间。web服务器收到请求后发现有头If-Modified-Since 则与被请求资源的最后修改时间进行比对。若最后修改时间较新，说明资源又被改动过，则响应整片资源内容（写在响应消息包体内），HTTP 200；若最后修改时间较旧，说明资源无新修改，则响应HTTP 304 (无需包体，节省浏览)，告知浏览器继续使用所保存的cache。

#### 4.Etag/If-None-Match
这个也需要配合Cache-Control使用
Etag 对应请求的资源在服务器中的唯一标识（具体规则由服务器决定），比如一张图片，它在服务器中的标识为 ETag: W/”ACXbWXd1n0CGMtAd65PcoA==”。

If-None-Match 如果浏览器在 Cache-Control:max-age=60 设置的时间超时后，发现消息头中还设置了Etag值。然后，浏览器会再次向服务器请求数据并添加 In-None-Match 消息头，它的值就是之前Etag值。服务器通过Etag来定位资源文件，根据它是否更新的情况给浏览器返回200或者是304。

Etag机制比Last-Modified精确度更高，如果两者同时设置的话，Etag优先级更高。

#### 5.Pragma
Pragma 头域用来包含实现特定的指令，最常用的是 Pragma:no-cache。
在HTTP/1.1协议中，它的含义和 Cache- Control:no-cache 相同。

## 使用缓存流程
![使用缓存](http://mmbiz.qpic.cn/mmbiz/zPh0erYjkib2D4WtKTwFjia6LOoTP8ZGh89NZDOkU7OnWqt1xYtNbIib9PZGyhprEYsX3ib6YJjcGmqHnReicriadx7Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

## OKHttpのCache
OKHTTP如果要设置缓存，首要的条件就是设置一个缓存文件夹
```java
private static final int DEFAULT_CACHE_SIZE = 1024 * 1024 * 20;
File cacheFile = new File(App.getApplication().getCacheDir(), "responses");
Cache cache = new Cache(cacheFile, DEFAULT_CACHE_SIZE);
OkHttpClient mOkHttpClient = new OkHttpClient.Builder()
                        .cache(cache)                        
                        .build();
```
如果服务器是支持缓存的，那么这样设置之后，再次访问的时候就可以读取缓存了。
"https://publicobject.com/helloworld.txt"，该地址就是支持缓存的。

其实控制缓存的消息头往往是服务端返回的信息中添加的如”Cache-Control:max-age=60”。
如果服务器不支持缓存，OKHTTP能够很轻易地处理这种情况。那就是定义一个拦截器，人为地添加Response中的消息头，然后再传递给用户，这样用户拿到的Response就有了我们理想当中的消息头Headers，从而达到控制缓存的意图，正所谓移花接木。

因为拦截器可以拿到 Request 和 Response，所以可以轻而易举地加工这些东西。在这里我们人为地添加 Cache-Control 消息头。
```java
private static final Interceptor RESPONSE_INTERCEPTOR = new Interceptor() {
    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request();
        Response response = chain.proceed(request);
        int maxAge;
        // 缓存数据
        // 控制缓存的最大生命时间
        if(!NetworkUtils.isConnected(App.getApplication())) {
            maxAge = DEFAULT_MAX_STALE_OFFLINE;
        } else {
            maxAge = DEFAULT_MAX_STALE_ONLINE;
        }
        return response.newBuilder()
                .removeHeader("Pragma") // 清除头信息，因为服务器如果不支持，会返回一些干扰信息，不清除下面无法生效
                .removeHeader("Cache-Control")
                .header("Cache-Control", "public, max-age=" + maxAge)
                .build();
    }
};

OkHttpClient mOkHttpClient = new OkHttpClient.Builder()
                        .cache(cache)   
                        .addNetworkInterceptor(RESPONSE_INTERCEPTOR)                     
                        .build();
```

#### 设置拦截器的缺点
网上有人说用拦截器进行缓存是野路子，是Hack行为。
不过用拦截器控制缓存确实也有一些不好的地方：
1. 网络访问请求的资源是文本信息，如新闻列表，这类信息经常变动，一天更新好几次，它们用的缓存时间应该就很短。
2. 网络访问请求的资源是图片或者视频，它们变动很少，或者是长期不变动，那么它们用的缓存时间就应该很长。

那么，问题来了。因为OKHTTP开发建议是同一个APP，用同一个OKHTTPCLIENT对象这是为了只有一个缓存文件访问入口。这个很容易理解，单例模式嘛。但是问题拦截器是在OKHttpClient.Builder当中添加的。如果在拦截器中定义缓存的方法会导致图片的缓存和新闻列表的缓存时间是一样的，这显然是不合理。

#### okhttp官方文档建议缓存方法
okhttp中建议用 CacheControl 这个类来进行缓存策略的制定。它内部有两个很重要的静态实例。
```java
/**
 * Cache control request directives that require network validation of responses. Note that such
 * requests may be assisted by the cache via conditional GET requests.
 */
public static final CacheControl FORCE_NETWORK = new Builder().noCache().build();

/**
 * Cache control request directives that uses the cache only, even if the cached response is
 * stale. If the response isn't available in the cache or requires server validation, the call
 * will fail with a {@code 504 Unsatisfiable Request}.
 */
public static final CacheControl FORCE_CACHE = new Builder()
    .onlyIfCached()
    .maxStale(Integer.MAX_VALUE, TimeUnit.SECONDS)
    .build();
```
我们看到 FORCE_NETWORK 常量 用来强制使用网络请求。FORCE_CACHE 只取本地的缓存。它们本身都是 CacheControl 对象，由内部的Buidler对象构造。下面我们来看看CacheControl.Builder

知道了 CacheControl 的相关信息，那么它怎么使用呢？不同于拦截器设置缓存， CacheControl 是针对 Request 的，所以它可以针对每个请求设置不同的缓存策略。比如图片和新闻列表。下面代码展示如何用 CacheControl 设置一个60秒的超时时间。
```java
File cacheFile = new File(getCacheDir(), "cache");
int cacheSize = 10 * 1024 * 1024;
final Cache cache = new Cache(cacheFile, cacheSize);
new Thread(new Runnable() {
    @Override
    public void run() {
        OkHttpClient okHttpClient = new OkHttpClient.Builder()
                .cache(cache)
                .build();

        CacheControl cacheControl = new CacheControl.Builder()
                .maxAge(60, TimeUnit.SECONDS)
                .build();
        Request request = new Request.Builder()
                .url(url)
                .cacheControl(cacheControl)
                .build();
        Response response = null;
        try {
            response = okHttpClient.newCall(request).execute();
            response.body().close();;
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}).start();
```
强制使用缓存:
```java
Request request = new Request.Builder()
        .url(url)
        .cacheControl(Cache.FORCE_CACHE)
        .build();
```
如果缓存不符合条件会返回504.这个时候我们要根据情况再进行编码，如缓存不行就再进行一次网络请求。
```java
Response response = okHttpClient.newCall(request).execute();
if(response.code() != 504) {
    // 资源已缓存，可以直接使用
} else {
    // 没有资源缓存，需要再去联网请求
}
```
不使用缓存
前面也有讲CacheControl.FORCE_NETWORK这个常量。
```java
Request request = new Request.Builder()
        .url(url)
        .cacheControl(Cache.FORCE_NETWORK)
        .build();
```
还有一种情况将maxAge设置为0，也不会取缓存，直接走网络。
```java
CacheControl cacheControl = new CacheControl.Builder()
        .maxAge(0, TimeUnit.SECONDS)
        .build();
Request request = new Request.Builder()
        .url(url)
        .cacheControl(cacheControl)
        .build();
```
