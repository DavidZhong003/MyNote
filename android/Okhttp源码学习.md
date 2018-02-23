# 基本使用 #

无论是是post还是get请求,基本是如下4步:

1. 建立一个OkHttpClient对象(包含协议路由,拦截器等信息)。
2. 创建Request对象，指定请求url和参数等。
3. 通过client创建call对象（newCall方法）
4. call执行同步或者异步方法


示例代码：

    OkHttpClient client = new OkHttpClient();
    RequestBody body = RequestBody.create(JSON, json);
    Request request = new Request.Builder()
     .url(url)
     .post(body)
     .build();
    Response response = client.newCall(request).execute());

# 基本流程 #

源码分析:

1. client.newcall方法获取的是一个RealCall对象。
	
	  /**
       * Prepares the {@code request} to be executed at some point in the future.
       */
     @Override public Call newCall(Request request) {
    	return new RealCall(this, request, false /* for web socket */);
      }

2.实际执行的是RealCall方法的execute方法

    @Override public Response execute() throws IOException {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    try {
      client.dispatcher().executed(this);
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } finally {
      client.dispatcher().finished(this);
    }
      }
    	 

其中逻辑是先判断是否被执行。没有执行调用client中的调度器执行，通过getResponseWithInterceptorChain方法获取响应结果。

3.getResponseWithInterceptorChain方法主要内容是

    Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));
    
    Interceptor.Chain chain = new RealInterceptorChain(
    interceptors, null, null, null, 0, originalRequest);
    return chain.proceed(originalRequest);
      } 

主要逻辑是，把自定义的拦截器以及内置的BridgeInterceptor、CacheInterceptor、ConnectInterceptor、CallServerInterceptor等拦截器添加进集合构建RealInterceptorChain对象再执行它的proceed方法。
这里给我们留下疑问：1、RealInterceptorChain是啥2、那些内置的拦截器的作用


4.RealInterceptorChain中proceed方法

    public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      Connection connection) throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();
    
    calls++;
    
    // If we already have a stream, confirm that the incoming request will use it.
    if (this.httpCodec != null && !sameConnection(request.url())) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
      + " must retain the same host and port");
    }
    
    // If we already have a stream, confirm that this is the only call to chain.proceed().
    if (this.httpCodec != null && calls > 1) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
      + " must call proceed() exactly once");
    }
    
    // Call the next interceptor in the chain.
    RealInterceptorChain next = new RealInterceptorChain(
    interceptors, streamAllocation, httpCodec, connection, index + 1, request);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);
    
    // Confirm that the next interceptor made its required call to chain.proceed().
    if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
      throw new IllegalStateException("network interceptor " + interceptor
      + " must call proceed() exactly once");
    }
    
    // Confirm that the intercepted response isn't null.
    if (response == null) {
      throw new NullPointerException("interceptor " + interceptor + " returned null");
    }
    
    return response;
      }

 去除中间判断逻辑,主要内容是构建next RealInterceptorChain(里面拦截器索引+1)对象,然后调用拦截器intercept的intercept方法


5.拦截器的intercept方法

Interceptor是接口,以一个它具体实现类BridgeInterceptor为例。intercept方法接受一个chain对象，也就是之前创建的RealInterceptorChain对象，里面逻辑是取出请求Request,经过处理后，传入chain中得到响应Response 然后经过处理后再返回。
整个流程就是**请求--拦截器1处理请求--拦截器2处理请求--.....--拦截器2处理响应--拦截器2处理响应--响应**，这是经典的责任链设计模式。

![okhttp流程图](https://upload-images.jianshu.io/upload_images/1639355-64b91013d44e15d5.png?imageMogr2/auto-orient/)

其中内置的拦截器有，

- RetryAndFollowUpInterceptor(重试，重定向拦截器)
- BridgeInterceptor（桥接拦截器，转换网络请求，添加头部信息）
- CacheInerceptor（缓存拦截器）
- ConnectInerceptor（连接拦截器，主要连接服务器，http，https包装）
- CallServerInterceptor（服务拦截器,主要负责数据发送，读取）


# OkHttpClient #

其中重要的就是Builder内部类
主要要构建的参数有：

    Dispatcher dispatcher;
	//代理 ， 直连接，http，socket
    Proxy proxy;
    List<Protocol> protocols;
    List<ConnectionSpec> connectionSpecs;
    final List<Interceptor> interceptors = new ArrayList<>();
    final List<Interceptor> networkInterceptors = new ArrayList<>();
    ProxySelector proxySelector;
    CookieJar cookieJar;
    Cache cache;
    InternalCache internalCache;
    SocketFactory socketFactory;
    SSLSocketFactory sslSocketFactory;
    CertificateChainCleaner certificateChainCleaner;
    HostnameVerifier hostnameVerifier;
    CertificatePinner certificatePinner;
    Authenticator proxyAuthenticator;
    Authenticator authenticator;
    ConnectionPool connectionPool;
    Dns dns;
    boolean followSslRedirects;
    boolean followRedirects;
    boolean retryOnConnectionFailure;
    int connectTimeout;
    int readTimeout;
    int writeTimeout;
    int pingInterval;

其中在Builder的构造函数内：

	public Builder() {
	  //请求分发器，内部是有ThreadPoolExecutor	
      dispatcher = new Dispatcher();
	  // 协议列表，默认为http1.1和http2.0	
      protocols = DEFAULT_PROTOCOLS;
	  //连接规范列表（TLS等）	
      connectionSpecs = DEFAULT_CONNECTION_SPECS;
	  //线路选择器	
      proxySelector = ProxySelector.getDefault();
      cookieJar = CookieJar.NO_COOKIES;
      socketFactory = SocketFactory.getDefault();
      hostnameVerifier = OkHostnameVerifier.INSTANCE;
      certificatePinner = CertificatePinner.DEFAULT;
      proxyAuthenticator = Authenticator.NONE;
      authenticator = Authenticator.NONE;
      connectionPool = new ConnectionPool();
	  //DNS解析	
      dns = Dns.SYSTEM;
      followSslRedirects = true;
      followRedirects = true;
      retryOnConnectionFailure = true;
      connectTimeout = 10_000;
      readTimeout = 10_000;
      writeTimeout = 10_000;
      pingInterval = 0;
    }

# Request类 #
 也是采用Builder设计模式，好处是可以把复杂对象的构建过程隐藏，解耦和拓展方便。
  	
	HttpUrl url;
    String method;
    Headers.Builder headers;
    RequestBody body;
    Object tag;

    public Builder() {
      this.method = "GET";
      this.headers = new Headers.Builder();
    }

主要是请求url，请求方法，请求头，请求体的构建

# RealCall #

实际访问类。实现Call接口，而Call接口主要方法有

- request（）返回Request请求对象
- execute同步执行方法
- enqueue异步回调方法
- cancel终止方法
- isExecute判断是否执行
- isCanceled判断是否终止
- clone克隆方法

而其实现类RealCall中重要的方法就是execute和enqueue，同步、异步获取响应。

同步的情况在上面分析过，下面重点解析异步情况

	@Override public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
    }

由上面的代码逻辑主要是判断这个call是否被执行，如果被执行抛出异常，然后调用方法captureCallStackTrace()，这个方法主要追踪调用信息，实际上把回调包装成AsyncCall对象交给Dispatcher执行，而实际上AsyncCall是Runnable对象，而dispatcher里面是线程池，执行run方法时调用AsyncCall里的execute方法。
	
	@Override protected void execute() {
      boolean signalledCallback = false;
      try {
        Response response = getResponseWithInterceptorChain();
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

里面逻辑一样，调用getResponseWithInterceptorChain获取响应结果，然后进行回调

# 拦截器分析 #

## RetryAndFollowUpInterceptor ##
重试及重定向拦截器。重点方法intercept（）

（源码略）

主要逻辑是



- 构建StreamAllocation对象（传递连接池connectionPool以及地址createAddress）。

	 	streamAllocation = new StreamAllocation(
        client.connectionPool(), createAddress(request.url()), callStackTrace);
- 开启while 死循环
- 调用下个拦截器获取响应结果
	`response = ((RealInterceptorChain) chain).proceed(request, streamAllocation, null, null);`

- 异常捕获，判断请求是否进行
- 判断是否进行重定向
	`Request followUp = followUpRequest(response);`
- 关闭响应
	`closeQuietly(response.body());`

而主要重试及重定向在followUpRequest中完成，处理的返回码为：

- 407（需要代理授权）
- 401(未授权)
- 307，308 临时重定向
- 300(多种选择)，301(永久移动)，302(临时移动)，303(查看其他位置)

## BridgeInterceptor ##
桥接拦截器，主要功能：

- 处理请求头，若没设置则使用默认配置
- 调用下个拦截器
- 对响应进行gzip，header，cookie处理


## ConnectInterceptor ##
连接拦截器

    @Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    StreamAllocation streamAllocation = realChain.streamAllocation();
    
    // We need the network to satisfy this request. Possibly for validating a conditional GET.
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    HttpCodec httpCodec = streamAllocation.newStream(client, doExtensiveHealthChecks);
    RealConnection connection = streamAllocation.connection();
    
    return realChain.proceed(request, streamAllocation, httpCodec, connection);
      }

里面代码少，但逻辑较为复杂。主要是获取第一个拦截器（RetryAndFollowUpInterceptor）创建的StreamAllocation对象，然后调用它的newStream方法，里面进行创建连接等操作，然后获取连接对象（RealConnection）传递到下个拦截器。
其中主要方法`streamAllocation.newStream(client, doExtensiveHealthChecks);`主要涉及的类有，StreamAllocation（allocation：分配）、ConnectionPool、RealConnection

1.`streamAllocation.newStream(client, doExtensiveHealthChecks);`方法返回一个HttpCodec，http编解码器，主要是RealConnection中newCodec方法获取，返回Http1Codec或者Http2Codec

里面重要逻辑调用获取健康的连接方法`RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
  writeTimeout, connectionRetryEnabled, doExtensiveHealthChecks);`
而findHealthyConnection方法主要是调用findConnection方法。

2.findConnection方法
  该方法里面逻辑较为复杂，梳理为：当前是否有连接（this.connection对象），有返回，没有则查看连接池（this.connectionPool）是否有连接（`Internal.instance.get(connectionPool, address, this);`里面从连接池中根据address获取），有返回，没有创建一个连接（`result = new RealConnection(connectionPool, selectedRoute);`对象），让其与当前连接关联（调用acquire方法），再添加改连接进连接池等操作

  而真正的连接方法是调用RealConnection对象的connect方法
	
	 // Do TCP + TLS handshakes. This is a blocking operation.
    result.connect(connectTimeout, readTimeout, writeTimeout, connectionRetryEnabled);
    routeDatabase().connected(result.route());`
	
	
3.RealConnection的connect方法
省略部分代码，核心代码是

		...
		if (route.requiresTunnel()) {
          connectTunnel(connectTimeout, readTimeout, writeTimeout);
        } else {
          connectSocket(connectTimeout, readTimeout);
        }
		...

如果是https则调用connectTunnel连接通道方法，不是则连接socket

3.1 connectSocket方法
	
	private void connectSocket(int connectTimeout, int readTimeout) throws IOException {
    Proxy proxy = route.proxy();
    Address address = route.address();

    rawSocket = proxy.type() == Proxy.Type.DIRECT || proxy.type() == Proxy.Type.HTTP
        ? address.socketFactory().createSocket()
        : new Socket(proxy);

    rawSocket.setSoTimeout(readTimeout);
    try {
      Platform.get().connectSocket(rawSocket, route.socketAddress(), connectTimeout);
    } catch (ConnectException e) {
      ConnectException ce = new ConnectException("Failed to connect to " + route.socketAddress());
      ce.initCause(e);
      throw ce;
    }
    source = Okio.buffer(Okio.source(rawSocket));
    sink = Okio.buffer(Okio.sink(rawSocket));
  	}

	里面获取socket对象用OkIo保存输入输出流
3.2 connectTunnel方法
里面也嗲用connectSocket方法，但多了创建隧道方法（`createTunnel(readTimeout, writeTimeout, tunnelRequest, url);`）

（todo 理解隧道）

回到拦截器中`streamAllocation.connection();`就是返回上面提到的RealConnection


