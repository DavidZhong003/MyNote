# 前言

[Retrofit](https://github.com/square/retrofit)是个极其优秀的库,特别是和rxjava结合起来，使用起来那是一个丝滑般爽。不过使用了一两年，现在才好好想起总结总结，下面直接用实际项目中的部分代码来进行分析Retrofit帮我们做了什么。
## 简单的使用
1. 创建Retrofit
```kotlin
        Retrofit.Builder().baseUrl(BuildConfig.HOST)
                .client(OKHTTP_CLIENT_NO_TOKEN)
                .addConverterFactory(GsonConverterFactory.create())
                .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
                .build()
```
2. 创建含有网络请求信息的ApiService
3. 获取ApiService
`fun getApiService(): ApiService = RETROFIT.create(ApiService::class.java)`
4. 调用ApiService 接口方法

# Builder 过程
Retrofit 通过 Builder 模式创建：
先来直接看 Builder.build方法：
```java
    public Retrofit build() {
      if (baseUrl == null) {
        throw new IllegalStateException("Base URL required.");
      }

      okhttp3.Call.Factory callFactory = this.callFactory;
      if (callFactory == null) {
        callFactory = new OkHttpClient();
      }

      Executor callbackExecutor = this.callbackExecutor;
      if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
      }

      // Make a defensive copy of the adapters and add the default Call adapter.
      List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
      adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

      // Make a defensive copy of the converters.
      List<Converter.Factory> converterFactories = new ArrayList<>(this.converterFactories);

      return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
          callbackExecutor, validateEagerly);
    }
```    
这是标准的build函数，检测一些可配置参数，没有设置就是赋默认值，然后创建目标对象。
## 创建Retrofit对象6个参数
这里创建Retrofit需要6个参数：
### 1. callFactory：okhttp3.Call.Factory
这里是我们传入的OkHttpClient 对象，如果我们没有传入，build方法会为我们创建一个默认的OkhttpClient对象。
### 2. baseUrl：HttpUrl
我们传入host字符串后会经过`HttpUrl.parse(baseUrl);`处理后转换为HttpUrl对象。
###  3. converterFactories：List<Converter.Factory>
1. Converter.Factory 是什么？。
CallAdapter.Factory是个抽象类：
里面有三个方法：
   - responseBodyConverter（）返回Converter<ResponseBody, ?>对象（负责把ResponseBody对象转为其他对象）
   - requestBodyConverter（）返回Converter<?, RequestBody>对象（负责把其他对象转为RequestBody）
   - stringConverter（）返回Converter<?,String>对象（把其他对象转为String对象）
   
   其中Converter<F,T>对象是个转换接口，里面convert方法，负责把F转换为T对象。   

2. converterFactories 里面有哪些对象？

   1. BuiltInConverters 对象
   在builder的构造方法里面创建了BuiltInConverters对象并添加到集合里面
   
   ```java
   Builder(Platform platform) {
      this.platform = platform;
      // Add the built-in converter factory first. This prevents overriding its behavior but also
      // ensures correct behavior when using converters that consume all types.
      converterFactories.add(new BuiltInConverters());
    }
   ```
   
   2. GsonConverterFactory 对象
   在上面配置中我们添加了GsonConverterFactory.create() 实际是GsonConverterFactory 对象

### 4. adapterFactories：List<CallAdapter.Factory>
1. CallAdapter.Factory 是什么？
一个抽象类，里面也是3个方法
a. get():CallAdapter 返回一个CallAdapter 适配器对象（有两个方法，一返回Type，二把Call转为其他对象的adapter方法）
b. getParameterUpperBound（） ：Type 返回参数上限类型
c. getRawType 获取原始类型

2. adapterFactories里面对象是什么？
   a. RxJava2CallAdapterFactory 对象
  在前面配置中`.addCallAdapterFactory(RxJava2CallAdapterFactory.create())`实际是RxJava2CallAdapterFactory对象
   b. ExecutorCallAdapterFactory 对象
  在build方法中会执行`adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));` ，里面实际创建的是ExecutorCallAdapterFactory对象。
    
###  5. callbackExecutor：Executor
回调执行者。
```java
      Executor callbackExecutor = this.callbackExecutor;
      if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
      }
```
而platform变量在builder构造方法中被初始化，Android平台下是Android对象，而platform.defaultCallbackExecutor();获得的是MainThreadExecutor对象

```java
  static class Android extends Platform {
    @Override public Executor defaultCallbackExecutor() {
      return new MainThreadExecutor();
    }

    @Override CallAdapter.Factory defaultCallAdapterFactory(Executor callbackExecutor) {
      return new ExecutorCallAdapterFactory(callbackExecutor);
    }

    static class MainThreadExecutor implements Executor {
      private final Handler handler = new Handler(Looper.getMainLooper());

      @Override public void execute(Runnable r) {
        handler.post(r);
      }
    }
  }
}
```

### 6. validateEagerly：boolean
默认为false

# create 方法做了什么
在刚开始用Retrofit时候一直很好奇，我们创建ApiService接口，没有创建任何实现对象，经过Create方法就可以直接调用接口的方法。
它这个方法里面做了什么呢。
来看下create的代码：

```java
  public <T> T create(final Class<T> service) {
  //验证是否是接口
    Utils.validateServiceInterface(service);
    //前面传入的参数，为false
    if (validateEagerly) {
     // 如果为true，会执行eagerlyValidateMethods方法，
     //里面会遍历接口的方法，创建对应的ServiceMethod对象并加到换成中
      eagerlyValidateMethods(service);
    }
    // 关键点！！！创建动态代理对象
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, Object[] args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
  }
```
这个方法主要分3部分，第一部分验证传入的class是否是接口，第二部分根据builder中参数是否预先创建ServiceMethod对象，而第三部分是最为重要的，创建动态代理对象。
## 动态代理Proxy
理解动态代理的同学，这部分可以跳过。。。。
简单的介绍下动态代理，利用反射技术在运行时候创建给定接口的动态代理对象实例。而创建这种代理对象也很简单，
Proxy.newProxyInstance()

里面接受3个参数：
1. ClassLoader 类加载器
2. Class<?>[] interfaces 接口class 数组
3. InvocationHandler 接口。

invoke 方法有3个参数：

1. proxy：Object 动态代理对象，可用来获取类信息，方法，annotation等
2. method：Method 被代用的方法，可获取方法名，参数，返回类型等信息
3. args：Object[] 方法参数


而动态代理类的方法都会交给实现了InvocationHandler接口对象的invoke（）处理。所以我们使用Retrofit创建了ApiService 代理对象。然后调用里面的方法时候其实是调用invoke方法

## Retrofit中create方法内InvocationHandler对象的invoke方法

上面我们已经搞起动态代理是什么东西，而Retrofit之所以能帮我们实现接口定义的方法就是因为这里面的invoke。

```java
          @Override public Object invoke(Object proxy, Method method, Object[] args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
```
前面两个判断是保证后面的执行的我们定义的方法。后面才是重点，
这里也分了3个步骤：
1. 根据method 加载得到ServiceMethod
2. 创建 OkHttpCall 对象。
3. 调用 serviceMethod.callAdapter.adapt(okHttpCall)

### ServiceMethod 是什么？

ServiceMethod 其实是个适配器，把我们接口定义的方法转为 http call。
里面除了一个Builder类还有3个重要的方法：
1. toRequest（）:Request  生成请求
2. toResponse():Response 生成响应
3. parsePathParameters():Set<String> 解析路径参数

先来看下serviceMethod怎么得到的，在invoke中调用了loadServiceMethod方法

```java
  ServiceMethod<?, ?> loadServiceMethod(Method method) {
    ServiceMethod<?, ?> result = serviceMethodCache.get(method);
    if (result != null) return result;

    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        result = new ServiceMethod.Builder<>(this, method).build();
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }
```

上面可看到，如果缓存中有就直接使用，没有就通过`new ServiceMethod.Builder<>(this, method).build();`创建。

#### ServiceMethod 的创建

先来分析下ServiceMethod的创建过程，上文知道先创建Builder对象然后调用build方法。

1. Builder构造

```java
    Builder(Retrofit retrofit, Method method) {
      this.retrofit = retrofit;
      this.method = method;
      this.methodAnnotations = method.getAnnotations();
      this.parameterTypes = method.getGenericParameterTypes();
      this.parameterAnnotationsArray = method.getParameterAnnotations();
    }
```
这个构造方法做的事情比较简单，就五个成员变量赋值，分别是Retrofit对象，方法对象，方法注解，参数类型，参数注解（二维数组）
2. build 方法

这个方法比较长，我们省略一些异常步骤：

```java
    public ServiceMethod build() {
      // 获取callAdapter
      callAdapter = createCallAdapter();
      // 获取相应类型
      responseType = callAdapter.responseType();
      if (responseType == Response.class || responseType == okhttp3.Response.class) {
        throw methodError("'"
            + Utils.getRawType(responseType).getName()
            + "' is not a valid response body type. Did you mean ResponseBody?");
      }
      // 赋值响应转换器
      responseConverter = createResponseConverter();
      // 解析方法注解
      for (Annotation annotation : methodAnnotations) {
        parseMethodAnnotation(annotation);
      }
      ...
      确保请求方式，请求体等信息正确
      ...
      // 方法参数数量
      int parameterCount = parameterAnnotationsArray.length;
      // 参数交给 ParameterHandler处理
      parameterHandlers = new ParameterHandler<?>[parameterCount];
      for (int p = 0; p < parameterCount; p++) {
        Type parameterType = parameterTypes[p];
        if (Utils.hasUnresolvableType(parameterType)) {
          throw parameterError(p, "Parameter type must not include a type variable or wildcard: %s",
              parameterType);
        }

        Annotation[] parameterAnnotations = parameterAnnotationsArray[p];
        if (parameterAnnotations == null) {
          throw parameterError(p, "No Retrofit annotation found.");
        }

        parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
      }
      ...
      确保请求参数信息正确
      ...
      // 创建 ServiceMethod 对象
      return new ServiceMethod<>(this);
    }
```
来大概过一下这个build方法，先获取callAdapter->再通过Retrofit获取Converter<Response,?>转换器->解析方法注解获取请求方式->解析方法参数获取请求参数->构成ServiceMethod对象。

2.1 callAdapter 是什么对象？
createCallAdapter方法内最终通过`(CallAdapter<T, R>) retrofit.callAdapter(returnType, annotations);`获取callAdapter对象，最终调用Retrofit的nextCallAdapter方法：

```java
  public CallAdapter<?, ?> nextCallAdapter(CallAdapter.Factory skipPast, Type returnType,
      Annotation[] annotations) {
    checkNotNull(returnType, "returnType == null");
    checkNotNull(annotations, "annotations == null");
    //skipPast = null 所以是-1+1 = 0
    int start = adapterFactories.indexOf(skipPast) + 1;
    
    for (int i = start, count = adapterFactories.size(); i < count; i++) {
      CallAdapter<?, ?> adapter = adapterFactories.get(i).get(returnType, annotations, this);
      if (adapter != null) {
        return adapter;
      }
    }
```
这里会返回adapterFactories集合中第一个适配器工厂类返回的适配器，RxJava2CallAdapterFactory（如果没有添加rxjava那就是默认的工厂类 ExecutorCallAdapterFactory 类），查看RxJava2CallAdapterFactory对象的get方法，发现返回的是RxJava2CallAdapter对象。

所以callAdapter是**RxJava2CallAdapter**对象（如果没添加rxjava，使用默认，这个对象是CallAdapter匿名子类）。

2.2 responseConverter 是什么对象？

同样ServiceMethod的 createResponseConverter 方法最终会调用 Retrofit的responseBodyConverter方法
`retrofit.responseBodyConverter(responseType, annotations);`
最终调用nextResponseBodyConverter方法：

```java
  public <T> Converter<ResponseBody, T> nextResponseBodyConverter(Converter.Factory skipPast,
      Type type, Annotation[] annotations) {
    checkNotNull(type, "type == null");
    checkNotNull(annotations, "annotations == null");

    int start = converterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = converterFactories.size(); i < count; i++) {
      Converter<ResponseBody, ?> converter =
          converterFactories.get(i).responseBodyConverter(type, annotations, this);
      if (converter != null) {
        //noinspection unchecked
        return (Converter<ResponseBody, T>) converter;
      }
    }
    ....
    ....
  }
```
这里的逻辑和nextCallAdapter类似，不过converterFactories中的第一个对象是BuiltInConverters，它的responseBodyConverter方法当方法返回类型不是ResponseBody或Void时候返回空的Converter对象。

```java
public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations,
      Retrofit retrofit) {
    if (type == ResponseBody.class) {
      return Utils.isAnnotationPresent(annotations, Streaming.class)
          ? StreamingResponseBodyConverter.INSTANCE
          : BufferingResponseBodyConverter.INSTANCE;
    }
    if (type == Void.class) {
      return VoidResponseBodyConverter.INSTANCE;
    }
    return null;
  }
```
所以我们看下GsonConverterFactory ：

```java
  @Override
  public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations,
      Retrofit retrofit) {
    TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
    return new GsonResponseBodyConverter<>(gson, adapter);
  }
```
所以我们等下用**GsonResponseBodyConverter**分析。

2.3 解析方法 注解

主要是parseMethodAnnotation方法：

```java
  private void parseMethodAnnotation(Annotation annotation) {
      ....
      else if (annotation instanceof GET) {
        parseHttpMethodAndPath("GET", ((GET) annotation).value(), false);
      } 
      ....
    }
```
里面对不同类型注解进行处理，最终都是交给parseHttpMethodAndPath方法，我们以GET为例子。

```
 private void parseHttpMethodAndPath(String httpMethod, String value, boolean hasBody) {
      ...
      ...
      this.relativeUrl = value;
      this.relativeUrlParamNames = parsePathParameters(value);
    }
```
其实里面是获取注解内的信息然后存储在成员变量中

2.4 解析方法参数注解

主要是生成 parameterHandler，`parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);`

而parseParameter方法最终调用parseParameterAnnotation方法生成 ParameterHandler

而ParameterHandler是个抽象类，不同注解有不同实现

### OkHttpCall 对象

构造器里面接受ServiceMethod 与 方法参数

### RxJava2CallAdapter 的 adapter 方法
最后create方法返回的是`serviceMethod.callAdapter.adapt(okHttpCall)` 对象

```java
  @Override public Object adapt(Call<R> call) {
    Observable<Response<R>> responseObservable = isAsync
        ? new CallEnqueueObservable<>(call)
        : new CallExecuteObservable<>(call);

    Observable<?> observable;
    if (isResult) {
      observable = new ResultObservable<>(responseObservable);
    } else if (isBody) {
      observable = new BodyObservable<>(responseObservable);
    } else {
      observable = responseObservable;
    }

    if (scheduler != null) {
      observable = observable.subscribeOn(scheduler);
    }

    if (isFlowable) {
      return observable.toFlowable(BackpressureStrategy.LATEST);
    }
    if (isSingle) {
      return observable.singleOrError();
    }
    if (isMaybe) {
      return observable.singleElement();
    }
    if (isCompletable) {
      return observable.ignoreElements();
    }
    return observable;
  }
```
这里返回不同的Rx 被观察者对象 ，我们以CallExecuteObservable为例看看里面怎么执行的

#### CallExecuteObservable 订阅方法

熟悉Rxjava的同学都知道，当Observable被订阅时候，会执行Observable的subscribeActual方法，我先来看下CallExecuteObservable的subscribeActual方法：

```java
 @Override protected void subscribeActual(Observer<? super Response<T>> observer) {
    // Since Call is a one-shot type, clone it for each new observer.
    // okhttpCall 对象
    Call<T> call = originalCall.clone();
    observer.onSubscribe(new CallDisposable(call));

    boolean terminated = false;
    try {
      // 重要步骤，发送网络请求获取响应
      Response<T> response = call.execute();
      if (!call.isCanceled()) {
        observer.onNext(response);
      }
      if (!call.isCanceled()) {
        terminated = true;
        observer.onComplete();
      }
    } catch (Throwable t) {
      Exceptions.throwIfFatal(t);
      if (terminated) {
        RxJavaPlugins.onError(t);
      } else if (!call.isCanceled()) {
        try {
          observer.onError(t);
        } catch (Throwable inner) {
          Exceptions.throwIfFatal(inner);
          RxJavaPlugins.onError(new CompositeException(t, inner));
        }
      }
    }
  }
```
可以看到里面有个重要的步骤`  Response<T> response = call.execute();`
通过OkhttpCall的execute方法获取响应然后进行后面Rx的onNext等步骤。

#### OkhttpCall 的execute 方法

来看下实际执行网络请求：

```java
 @Override public Response<T> execute() throws IOException {
    okhttp3.Call call;

    synchronized (this) {
      if (executed) throw new IllegalStateException("Already executed.");
      executed = true;

      if (creationFailure != null) {
        if (creationFailure instanceof IOException) {
          throw (IOException) creationFailure;
        } else {
          throw (RuntimeException) creationFailure;
        }
      }

      call = rawCall;
      if (call == null) {
        try {
          call = rawCall = createRawCall();
        } catch (IOException | RuntimeException e) {
          creationFailure = e;
          throw e;
        }
      }
    }

    if (canceled) {
      call.cancel();
    }

    return parseResponse(call.execute());
  }
```
 可以看到里面其实分3个工作：
 1. 创建Okhttp3.Call
 2. 执行execute获取响应
 3. 执行parseResponse 转换响应
 
而其中execute获取响应是调用Okhttp中的方法，所以我们来分析下其他两个步骤：

- createRawCall 得到okhttp.Call

```java
  private okhttp3.Call createRawCall() throws IOException {
    // 获取请求
    Request request = serviceMethod.toRequest(args);
    // 其实是OkhttpClient.newCall()方法
    okhttp3.Call call = serviceMethod.callFactory.newCall(request);
    if (call == null) {
      throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
  }
```
终于看见关键步骤了，这方法里先通过serviceMethod 获取请求，而service.callFactory是调用Retrofit中的callFactory，也就是我们build中传入的OkhttpClient对象，所以其实Retrofit还是通过Okhttp进行网络请求。

- 执行parseResponse

前面已经获取了原始的响应了，怎么获取

```java

  Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    ResponseBody rawBody = rawResponse.body();

    // Remove the body's source (the only stateful object) so we can pass the response along.
    // 让响应source可传递
    rawResponse = rawResponse.newBuilder()
        .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
        .build();
    // 获取响应码
    int code = rawResponse.code();
    // 异常响应
    if (code < 200 || code >= 300) {
      try {
        // Buffer the entire body to avoid future I/O.
        ResponseBody bufferedBody = Utils.buffer(rawBody);
        return Response.error(bufferedBody, rawResponse);
      } finally {
        rawBody.close();
      }
    }
    // 204,205 响应成功，无数据
    if (code == 204 || code == 205) {
      rawBody.close();
      return Response.success(null, rawResponse);
    }

    ExceptionCatchingRequestBody catchingBody = new ExceptionCatchingRequestBody(rawBody);
    try {
      // 获取响应，把响应体的包装类通过serviceMethod处理
      T body = serviceMethod.toResponse(catchingBody);
      return Response.success(body, rawResponse);
    } catch (RuntimeException e) {
      // If the underlying source threw an exception, propagate that rather than indicating it was
      // a runtime exception.
      catchingBody.throwIfCaught();
      throw e;
    }
  }
```
上面分析可以看到原始的响应体经过`serviceMethod.toResponse(catchingBody);`处理变成我们想要的数据。然后把得到的数据包装成Response（并非ok中的Response）对象。

而ServiceMethod中的toResponse也很简单：

```java
  R toResponse(ResponseBody body) throws IOException {
    return responseConverter.convert(body);
  }
```
而responseConverter对象在之前分析了是GsonResponseBodyConverter,所以我们看下它的convert方法：

```java
  @Override public T convert(ResponseBody value) throws IOException {
    JsonReader jsonReader = gson.newJsonReader(value.charStream());
    try {
      return adapter.read(jsonReader);
    } finally {
      value.close();
    }
  }
```
这就是可以得到们要想对bean类的原因。


# 总结

至此，我们Retrofit的流程分析的差不多了。现在来总结下流程：

1. 通过Retrofit.Builder 生成Retrofit对象（主要传入，baseUrl，OkhttpClient，coverter，callAdapter等）。
2. 调用create方法得到接口的代理对象。
3. 执行接口的某个方法触发代理对象的invoke方法。
4. 通过Builder模式，生成 ServiceMethod 对象(期间，进行变量赋值，解析方法中注解path，解析请求参数等信息)
5. 生成OkhttpCall 对象
6. 执行callAdapter 中的 adapter 方法（后续以RxJava2CallAdapter为例）
7. 生成XXXObservable等被观察者对象（后续以CallExecuteObservable为例）
8. 下游订阅执行 subscribeActual 方法
9. 执行OkhttpCall方法生成响应（期间使用serviceMethod生成Okhttp的Request，经过Okhttpclien得到call对象，然后生成原始Response，然后使用serviceMethod中的converter转换原始响应得到Gson处理后的响应数据）
10. 完成一次请求。