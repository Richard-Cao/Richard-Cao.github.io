title: Retrofit2源码分析（一）
date: 2016-05-29 19:51:36
categories: 源码分析
tags: retrofit
---
转载请注明出处：http://richardcao.me/

> 本文将顺着**构建请求对象→构建请求接口→发起同步/异步请求**的流程，分析retrofit2是如何实现的。

## 组成部分
Retrofit2源码主要分为以下几个部分：

* retrofit
* retrofit-adapters
* retrofit-converters

本篇先分析retrofit部分，也就是retrofit源码的主干部分。

下面顺着retrofit2的使用流程进行分析。

## Retrofit对象的构建
首先我们看看构建一个Retrofit对象，都需要或者可选哪些配置：

```java
/**
   * Build a new {@link Retrofit}.
   * <p>
   * Calling {@link #baseUrl} is required before calling {@link #build()}. All other methods
   * are optional.
   */
  public static final class Builder {
  	//支持的平台
    private Platform platform;
    //发起请求的okhttp3的client工厂
    private okhttp3.Call.Factory callFactory;
    //okhttp3里的HttpUrl对象，用来解析和包装url
    private HttpUrl baseUrl;
    //转换器的工厂集合，retrofit构建可以插入多个转换器，比如gson转换器，Jackson转换器等
    private List<Converter.Factory> converterFactories = new ArrayList<>();
    //用来发起request和接收response的Call适配器，retrofit支持rxjava就是通过引入retrofit-adapters支持的，就是这个CallAdapter，这里先不作过多解释
    private List<CallAdapter.Factory> adapterFactories = new ArrayList<>();
    //Executor并发框架，包括线程池等
    private Executor callbackExecutor;
    //是否需要立即生效
    private boolean validateEagerly;

    Builder(Platform platform) {
      this.platform = platform;
      // Add the built-in converter factory first. This prevents overriding its behavior but also
      // ensures correct behavior when using converters that consume all types.
      converterFactories.add(new BuiltInConverters());
    }

    public Builder() {
      this(Platform.get());
    }

    ...
 }
```
Retrofit对象的构建使用了建造者模式，这里有一系列参数可以供我们选择，我给这些参数加了注释。这里构造方法中`Platform.get()`就是获取当前使用retrofit的平台信息，之前我用retrofit的时候，以为只支持Android平台，没想到还支持java8和ios，只不过这里的ios是指通过[robovm平台](https://robovm.com/)构建的ios程序，目前robovm主要的成功案例还是游戏，跨android & ios双平台。在Builder中，有一个比较重要的配置项，就是baseURL。我们设置baseUrl就是一个string字符串，retrofit是这么处理的：

```java
/**
     * Set the API base URL.
     * <p>
     * The specified endpoint values (such as with {@link GET @GET}) are resolved against this
     * value using {@link HttpUrl#resolve(String)}. The behavior of this matches that of an
     * {@code <a href="">} link on a website resolving on the current URL.
     * <p>
     * <b>Base URLs should always end in {@code /}.</b>
     * <p>
     * A trailing {@code /} ensures that endpoints values which are relative paths will correctly
     * append themselves to a base which has path components.
     * <p>
     * <b>Correct:</b><br>
     * Base URL: http://example.com/api/<br>
     * Endpoint: foo/bar/<br>
     * Result: http://example.com/api/foo/bar/
     * <p>
     * <b>Incorrect:</b><br>
     * Base URL: http://example.com/api<br>
     * Endpoint: foo/bar/<br>
     * Result: http://example.com/foo/bar/
     * <p>
     * This method enforces that {@code baseUrl} has a trailing {@code /}.
     * <p>
     * <b>Endpoint values which contain a leading {@code /} are absolute.</b>
     * <p>
     * Absolute values retain only the host from {@code baseUrl} and ignore any specified path
     * components.
     * <p>
     * Base URL: http://example.com/api/<br>
     * Endpoint: /foo/bar/<br>
     * Result: http://example.com/foo/bar/
     * <p>
     * Base URL: http://example.com/<br>
     * Endpoint: /foo/bar/<br>
     * Result: http://example.com/foo/bar/
     * <p>
     * <b>Endpoint values may be a full URL.</b>
     * <p>
     * Values which have a host replace the host of {@code baseUrl} and values also with a scheme
     * replace the scheme of {@code baseUrl}.
     * <p>
     * Base URL: http://example.com/<br>
     * Endpoint: https://github.com/square/retrofit/<br>
     * Result: https://github.com/square/retrofit/
     * <p>
     * Base URL: http://example.com<br>
     * Endpoint: //github.com/square/retrofit/<br>
     * Result: http://github.com/square/retrofit/ (note the scheme stays 'http')
     */
    public Builder baseUrl(HttpUrl baseUrl) {
      checkNotNull(baseUrl, "baseUrl == null");
      List<String> pathSegments = baseUrl.pathSegments();
      if (!"".equals(pathSegments.get(pathSegments.size() - 1))) {
        throw new IllegalArgumentException("baseUrl must end in /: " + baseUrl);
      }
      this.baseUrl = baseUrl;
      return this;
    }
```
这里的注释非常详细，讲解了baseUrl到底是什么，应该遵循什么样的格式，然后经过HttpUrl的解析，组合成okhttp3用来请求的url链接。这里规定了baseUrl末尾应该以`/`符号结尾，在后续API的接口类中后半部分定义应该不以`/`开头，这是和retrofit 1.x版本不同的地方。最后就是build方法了：

```java
/**
     * Create the {@link Retrofit} instance using the configured values.
     * <p>
     * Note: If neither {@link #client} nor {@link #callFactory} is called a default {@link
     * OkHttpClient} will be created and used.
     */
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
从这里可以看出如果我们不设置`callFactory`，retrofit会默认帮我们new一个OkHttpClient，如果我们不设置`callbackExecutor`，也会帮我们默认获取到当前平台默认的callbackExecutor，最后new一个Retrofit对象，到这里，Retrofit对象的构建就讲完了。这里有个值得注意的地方：CallAdapter和Converter的工厂集合都使用了保护性拷贝。那么保护性拷贝是什么呢？这是用来保证代码健壮性的。为什么要在这里用？因为Builder的这些配置的方法都是public的，虽然看起来这些是不可变的，但是可以通过传入构造参数来修改引用的值，这就会造成约束条件被破坏，所以使用了保护性拷贝来防止这种情况。

## API的编写
我们已经new好了一个我们需要的Retrofit对象，那么下一步就是编写API了。如何编写API呢？Retrofit的方式是用过java interface和注解的方式进行定义。例如：

```java
public interface GitHubService {
  @GET("users/{user}/repos")
  Call<List<Repo>> listRepos(@Path("user") String user);
}
```
这是官方文档上的一个例子。这里简单的定义了一个API，针对这小部分代码，我们来分析分析。
首先是GET注解，我们来看看这个注解是什么：

```java
/** Make a GET request. */
@Documented
@Target(METHOD)
@Retention(RUNTIME)
public @interface GET {
  /**
   * A relative or absolute path, or full URL of the endpoint. This value is optional if the first
   * parameter of the method is annotated with {@link Url @Url}.
   * <p>
   * See {@linkplain retrofit2.Retrofit.Builder#baseUrl(HttpUrl) base URL} for details of how
   * this is resolved against a base URL to create the full endpoint URL.
   */
  String value() default "";
}
```
这是一个运行时的方法注解，用来构造get请求，唯一的参数是一个string值，默认是空字符串。那么我们可以理解了，`@GET(xxxxx)`就是构造了一个用于get请求的url。下一个注解是Path注解，是一个运行时的参数注解，它是为了方便我们构建动态的url，参数是一个string值，还可以设置参数是否已经是URL encode编码，默认是false。最后我们看到，通过`Call<T>`构建成一个interface。`Call<T>`这个接口分别在`OkHttpCall`和`ExecutorCallbackCall`中做了具体的实现。

## 创建retrofit service
最最关键的一步来了。我们new好了retrofit对象，也写好了API interface，那怎么请求这个API呢，我们需要通过retrofit的`create`函数，创建一个service，然后调用API的接口方法进行请求并获得回传。使用当然很简单：

```java
GitHubService service = retrofit.create(GitHubService.class);
```
是的你没有看错，就这么一句话就搞定了。具体是怎么实现的呢？我们先来看看create的代码：

```java
@SuppressWarnings("unchecked") // Single-interface proxy creation guarded by parameter safety.
  public <T> T create(final Class<T> service) {
    Utils.validateServiceInterface(service);
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, Object... args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            ServiceMethod serviceMethod = loadServiceMethod(method);
            OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
  }
```
这里实现的非常巧妙，使用了java的动态代理。什么是动态代理？就是代理类、委托类都要实现同一个接口，然后代理通过接口代理委托类。由于java的单继承特性，所以动态代理是基于接口的，只能针对接口创建代理类。所以这里create的时候，先判断了一下传进来的service是不是interface，如果不是接口类型或者包含多个接口，就会直接抛异常。然后就是重头戏：动态代理的invoke方法。这个方法是`InvocationHandler`接口中唯一的方法，这个接口是代理实例实现的接口。代理实例调用方法时，`InvocationHandler `接口会将对方法的调用指派到代理的invoke方法中，进行处理。这里当method是一个对象的method时，直接调用。如果是平台的默认方法（根据Platform代码中这种情况是java8）就直接执行调用默认方法。正常在android平台下，会把method加载到`ServiceMethod`对象中，这里用了缓存，如果缓存中有`SeriveMethod`就直接取出，如果没有接new一个，然后用过`ServiceMethod`初始化了一个`OkHttpCall`对象，最后通过`callAdapter`的`adapt`方法返回一个代理了`Call`的实例。这个方法在这两个类中有具体的实现：
`DefaultCallAdapterFactory`和`ExecutorCallAdapterFactory`。这两个工厂类又在哪里会用到？就在Retrofit对象的build方法中：

```java
// Make a defensive copy of the adapters and add the default Call adapter.
      List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
      adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));
```
那我们再看看`platform.defaultCallAdapterFactory(callbackExecutor)`里是什么：

```java
CallAdapter.Factory defaultCallAdapterFactory(Executor callbackExecutor) {
    if (callbackExecutor != null) {
      return new ExecutorCallAdapterFactory(callbackExecutor);
    }
    return DefaultCallAdapterFactory.INSTANCE;
  }
```
可以看到如果在创建Retrofit对象时`callbackExecutor`为空的话，就new一个`ExecutorCallAdapterFactory `对象作为CallAdapter，如果不为空就返回`DefaultCallAdapterFactory`的实例。

## 发起请求
这里我们按照retrofit官方文档中的例子构建Retrofit对象。
retrofit2中，同步和异步不再用interface的定义区分，统一都为`Call<T>`，但是在Call接口的方法中有区分，我们来看看：

```java
/**
   * Synchronously send the request and return its response.
   *
   * @throws IOException if a problem occurred talking to the server.
   * @throws RuntimeException (and subclasses) if an unexpected error occurs creating the request
   * or decoding the response.
   */
  Response<T> execute() throws IOException;

  /**
   * Asynchronously send the request and notify {@code callback} of its response or if an error
   * occurred talking to the server, creating the request, or processing the response.
   */
  void enqueue(Callback<T> callback);
```
`execute()`是同步请求，`enqueue()`是异步请求。我们先看异步请求是如何实现的。enqueue这个方法在以下两个地方有实现：
`OkHttpCall`和`ExecutorCallbackCall`。那么当发起了异步请求之后，就会调用`ExecutorCallbackCall`中的`enqueue`方法，`ExecutorCallbackCall`中一个对象叫

```java
final Call<T> delegate;
```
这就是代理的Call对象，在这里就是我们在动态代理`create`方法中用过`ServiceMethod`对象构造的`OkHttpCall`对象，那么这里调用的enqueue方法，就是调用了代理的OkHttpCall对象的enqueue方法：

```java
@Override public void enqueue(final Callback<T> callback) {
    ... //各种判空等代码，这里先省略了

    okhttp3.Call call;
    Throwable failure;

    if (canceled) {
      call.cancel();
    }

    call.enqueue(new okhttp3.Callback() {
      @Override public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse)
          throws IOException {
        Response<T> response;
        try {
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
可以看到，这里其实是用了okhttp3的请求回调，做了一层封装，变成了retrofit2的Callback，也可以看出retrofit2和okhttp3是深度集成的。到这里，异步请求就一目了然了。我们还可以发现，retrofit2支持的可以取消请求，其实用的就是okhttp3的cancel方法。

同理，同步请求也是一样的，也是调用了代理的OkHttpCall的方法，只不过是`execute`方法，这个方法里面没有回调，和异步不同，是直接返回解析好的Response对象的，这就是同步请求啦。

至此，顺着**构建请求对象→构建请求接口→发起同步/异步请求**这个流程，我们分析了一遍retrofit2到底是如何实现的。

## 最后再说几句
* retrofit2主模块源码其实并不是很多，我感觉用的最巧妙的就是`create`方法的动态代理，然后加上运行时注解来构建API，深度结合okhttp3，使得网络请求的构建变得非常简洁，并且功能强大，而且安全。
* retrofit2同时支持与rxjava配合使用，是通过设置adapter来实现的，retrofit2把adapters和converters从主代码里拆分出来了，相当于组件化的意思，如果需要使用，就由开发者自己引用并定义，这种组件化的思想我觉得也非常棒。
* retrofit2源码里是通过junit来写测试的，测试代码写的也非常好，更说明了优秀的代码离不开优秀的测试，这也是值得我们学习的地方。

这是我第一次自己去分析一个开源项目的源码，也许可能会有遗漏的地方，望指出，抛砖引玉，谢谢~