## retrofit源码解析（一）
retrofit中是对okHttp的包装，采用了许多设计模式，笔者主要叙述以下三个
1. 动态代理模式

2. 适配器模式

3. 享元模式

   当然还有诸如外观模式、建造者模式、工厂模式等就不详述了


<center style="color:black;font-size:120%">动态代理模式模式</center>

```java
    Retrofit.Builder()
            .baseUrl(BASE_URL)
            .client(instance())
            .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
            .addConverterFactory(GsonConverterFactory.create())
            .build()
            .create(Api::class.java)
```
入口就在create函数
~~~java
public <T> T create(final Class<T> service) {
    Utils.validateServiceInterface(service);
    //预加载
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    //创建真实代理的对象，并返回
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            //解析method
            ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
            //创建网络请求对象
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
            //执行真实代理并返回真实的返回值，如Observable<User>
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
  }


~~~
retrofit并没有采用代码生成技术生成实现开发者自定义service接口的类，而是采用**动态代理**技术，代理了接口方法调用的真实操作

<center style="color:black;font-size:120%">适配器模式</center>

看一下serviceMethod类

~~~java
/** Adapts an invocation of an interface method into an HTTP call. */
final class ServiceMethod<R, T> {
  // Upper and lower characters, digits, underscores, and hyphens, starting with a character.
  static final String PARAM = "[a-zA-Z][a-zA-Z0-9_-]*";
  //匹配@GET、@POST等路径中的{}
  static final Pattern PARAM_URL_REGEX = Pattern.compile("\\{(" + PARAM + ")\\}");
  static final Pattern PARAM_NAME_REGEX = Pattern.compile(PARAM);
//采用了工厂方法模式，生产callAdapter
  final okhttp3.Call.Factory callFactory;
  //如RxJava2CallAdapter继承自callAdapter
  final CallAdapter<R, T> callAdapter;

  private final HttpUrl baseUrl;
  //如GsonResponseBodyConverter继承了Converter<ResponseBody, T>
  private final Converter<ResponseBody, R> responseConverter;
  private final String httpMethod;
  private final String relativeUrl;
  private final Headers headers;
  private final MediaType contentType;
  private final boolean hasBody;
  private final boolean isFormEncoded;
  private final boolean isMultipart;
  private final ParameterHandler<?>[] parameterHandlers;
......
}
~~~

~~~java
ServiceMethod<Object, Object> serviceMethod = 
    (ServiceMethod<Object, Object>) loadServiceMethod(method);
//创建网络请求对象
 OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
//执行真实代理并返回真实的返回值，如Observable<User>
 return serviceMethod.callAdapter.adapt(okHttpCall);
~~~

在构造出serviceMethod后，最后的真实代理交给了callAdapter（适配器）处理，在实际开发中，retrofit经常会和Rxjava、Gson一起使用，Retrofit采用的方式就是通过为Retrofit添加一个Rxjava的适配器，对请求Retrofit的call请求重新包装，在方法调用的时返回我们需要的类型，比如Observable、Flowable，而和Gson的配合使用也是通过在retrofit中添加一个适配器，把返回的数据转换成json
<center style="color:black;font-size:120%">享元模式</center>

进入loadServiceMethod

~~~java
  ServiceMethod<?, ?> loadServiceMethod(Method method) {
    ServiceMethod<?, ?> result = serviceMethodCache.get(method);
    if (result != null) return result;
    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
      //获取serviceMethod实例
        result = new ServiceMethod.Builder<>(this, method).build();
        //把实例保存在map中
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }
~~~


在这个方法中，采用了双重校验锁，在获取ServiceMehod的实例后，用map把实例存起来，已达到多次调用请求而不用重复加载ServiceMethod，共享第一次加载的实例的目的。

<center style="color:black;font-size:120%">走进源码</center>

简单的时序图

![](image\Retrofit.png)

ServiceMethod的是通过ServiceMethod.Builder构建的，进入Builder构造函数

~~~java
    Builder(Retrofit retrofit, Method method) {
      this.retrofit = retrofit;
      this.method = method;
      //获取方法级别的注解，如@GET、@POST，不包括@Path，@Field等在参数上的注解
      this.methodAnnotations = method.getAnnotations();
      //获取形式参数的类型，返回的是一个Type类型的数组
      this.parameterTypes = method.getGenericParameterTypes();
      //获取形式参数的注解，如@Path、@Query
      this.parameterAnnotationsArray = method.getParameterAnnotations();
    }

~~~

构造方法主要初始化几个参数，接下来进入build方法

~~~java

 public ServiceMethod build() {
 //重要，获取callAdapter
      callAdapter = createCallAdapter();
      //获得如Observable<User>中User的类型
      responseType = callAdapter.responseType();
      if (responseType == Response.class || responseType == okhttp3.Response.class) {
        throw methodError("'"

            + Utils.getRawType(responseType).getName()
                        + "' is not a valid response body type. Did you mean ResponseBody?");
                        }
                        //获取如GsonResponseConverter的responseConverter
                        responseConverter = createResponseConverter();
            //一个方法可能同时存在@POST、@Multipart多个注解，所以需要遍历
            for (Annotation annotation : methodAnnotations) {
            //解析方法级别的注解
                parseMethodAnnotation(annotation);
            }
            if (httpMethod == null) {
                throw methodError("HTTP method annotation is required (e.g., @GET, @POST, etc.).");
            }

      if (!hasBody) {
        if (isMultipart) {
          throw methodError(
              "Multipart can only be specified on HTTP methods with request body (e.g., @POST).");
        }
        if (isFormEncoded) {
          throw methodError("FormUrlEncoded can only be specified on HTTP methods with "
              + "request body (e.g., @POST).");
        }
      }
    
      int parameterCount = parameterAnnotationsArray.length;
      parameterHandlers = new ParameterHandler<?>[parameterCount];
      //遍历形式参数，参数注解
      for (int p = 0; p < parameterCount; p++) {
        Type parameterType = parameterTypes[p];
        //判断参数类型是否合法
        if (Utils.hasUnresolvableType(parameterType)) {
          throw parameterError(p, "Parameter type must not include a type variable or wildcard: %s",
              parameterType);
        }
    
        Annotation[] parameterAnnotations = parameterAnnotationsArray[p];
        if (parameterAnnotations == null) {
          throw parameterError(p, "No Retrofit annotation found.");
        }
        //解析参数中如@Path的注解，结果存到一个数组中
        parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
      }
    
      if (relativeUrl == null && !gotUrl) {
        throw methodError("Missing either @%s URL or @Url parameter.", httpMethod);
      }
      if (!isFormEncoded && !isMultipart && !hasBody && gotBody) {
        throw methodError("Non-body HTTP method cannot contain @Body.");
      }
      if (isFormEncoded && !gotField) {
        throw methodError("Form-encoded method must contain at least one @Field.");
      }
      if (isMultipart && !gotPart) {
        throw methodError("Multipart method must contain at least one @Part.");
      }
    
      return new ServiceMethod<>(this);
    }
~~~
这个方法囊括了很多信息
- CallAdapter的加载，也就是我们的适配器

- ResponseConverter的加载，适配器

- 对方法级别的注解的解析

- 对形式参数注解的解析

- 对用户请求的构建是否合理进行判断
下面进入createCallAdapter()方法
~~~java
 private CallAdapter<T, R> createCallAdapter() {
 //拿到返回类型，如Observable<User>
      Type returnType = method.getGenericReturnType();
      if (Utils.hasUnresolvableType(returnType)) {
        throw methodError(
            "Method return type must not include a type variable or wildcard: %s", returnType);
      }
      if (returnType == void.class) {
        throw methodError("Service methods cannot return void.");
      }
      Annotation[] annotations = method.getAnnotations();
      try {
        //noinspection unchecked
        //从retrofit中拿到callAdapter实例
        return (CallAdapter<T, R>) retrofit.callAdapter(returnType, annotations);
      } catch (RuntimeException e) { // Wide exception range because factories are user code.
        throw methodError(e, "Unable to create call adapter for %s", returnType);
      }
    }
~~~
这里主要是获取返回类型，并检查返回类型是否满足retrofit的要求，最后把获取callAdapter的工作交给了retrofit，进入retrofit.callAdapter(returnType, annotations);
~~~java
  public CallAdapter<?, ?> callAdapter(Type returnType, Annotation[] annotations) {
    return nextCallAdapter(null, returnType, annotations);
  }
~~~
进入nextCallAdapter
~~~java
public CallAdapter<?, ?> nextCallAdapter(@Nullable CallAdapter.Factory skipPast, Type returnType,
      Annotation[] annotations) {
    checkNotNull(returnType, "returnType == null");
    checkNotNull(annotations, "annotations == null");

    int start = adapterFactories.indexOf(skipPast) + 1;
    //遍历创建retrofit时存入的adapterFactories，如RxJava2CallAdapterFactory
    for (int i = start, count = adapterFactories.size(); i < count; i++) {
    //根据返回值和annotations、retrofit实例获取adapter
      CallAdapter<?, ?> adapter = adapterFactories.get(i).get(returnType, annotations, this);
      if (adapter != null) {
        return adapter;
      }
    }
    
    StringBuilder builder = new StringBuilder("Could not locate call adapter for ")
        .append(returnType)
        .append(".\n");
    if (skipPast != null) {
      builder.append("  Skipped:");
      for (int i = 0; i < start; i++) {
        builder.append("\n   * ").append(adapterFactories.get(i).getClass().getName());
      }
      builder.append('\n');
    }
    builder.append("  Tried:");
    for (int i = start, count = adapterFactories.size(); i < count; i++) {
      builder.append("\n   * ").append(adapterFactories.get(i).getClass().getName());
    }
    throw new IllegalArgumentException(builder.toString());
  }
~~~
这里把adapter的获取交给了factory，为了方便理解，采用RxJava2CallAdapterFactory来说明
~~~java
 @Override public @Nullable CallAdapter<?, ?> get(
      Type returnType, Annotation[] annotations, Retrofit retrofit) {
    Class<?> rawType = getRawType(returnType);

    if (rawType == Completable.class) {
      // Completable is not parameterized (which is what the rest of this method deals with) so it
      // can only be created with a single configuration.
      return new RxJava2CallAdapter(Void.class, scheduler, isAsync, false, true, false, false,
          false, true);
    }
    
    boolean isFlowable = rawType == Flowable.class;
    boolean isSingle = rawType == Single.class;
    boolean isMaybe = rawType == Maybe.class;
     //只能处理如Observable等返回类型，如果返回类型不能处理，则返回null
    if (rawType != Observable.class && !isFlowable && !isSingle && !isMaybe) {
      return null;
    }
......
   return new RxJava2CallAdapter(responseType, scheduler, isAsync, isResult, isBody, isFlowable,
        isSingle, isMaybe, false);
}
~~~

从这里可以看出，当方法的返回类型factory不能处理时，就返回null，交给下一个factory处理(如果有)。

回到ServiceMethod build() 方法，可以看到下一步时获取responseConveter了，而 responseConverter的获取和callAdapter的获取基本一致，这里就不阐述了，下面进入parseMethodAnnotation(annotation)方法

~~~java
 private void parseMethodAnnotation(Annotation annotation) {
      if (annotation instanceof DELETE) {
        parseHttpMethodAndPath("DELETE", ((DELETE) annotation).value(), false);
      } else if (annotation instanceof GET) {
        parseHttpMethodAndPath("GET", ((GET) annotation).value(), false);
      } else if (annotation instanceof HEAD) {
        parseHttpMethodAndPath("HEAD", ((HEAD) annotation).value(), false);
        if (!Void.class.equals(responseType)) {
          throw methodError("HEAD method must use Void as response type.");
        }
      } else if (annotation instanceof PATCH) {
        parseHttpMethodAndPath("PATCH", ((PATCH) annotation).value(), true);
      } else if (annotation instanceof POST) {
        parseHttpMethodAndPath("POST", ((POST) annotation).value(), true);
      } else if (annotation instanceof PUT) {
        parseHttpMethodAndPath("PUT", ((PUT) annotation).value(), true);
      } else if (annotation instanceof OPTIONS) {
        parseHttpMethodAndPath("OPTIONS", ((OPTIONS) annotation).value(), false);
      } else if (annotation instanceof HTTP) {
        HTTP http = (HTTP) annotation;
        parseHttpMethodAndPath(http.method(), http.path(), http.hasBody());
      } else if (annotation instanceof retrofit2.http.Headers) {
        String[] headersToParse = ((retrofit2.http.Headers) annotation).value();
        if (headersToParse.length == 0) {
          throw methodError("@Headers annotation is empty.");
        }
        headers = parseHeaders(headersToParse);
      } else if (annotation instanceof Multipart) {
        if (isFormEncoded) {
          throw methodError("Only one encoding annotation is allowed.");
        }
        isMultipart = true;
      } else if (annotation instanceof FormUrlEncoded) {
        if (isMultipart) {
          throw methodError("Only one encoding annotation is allowed.");
        }
        isFormEncoded = true;
      }
    }
~~~

这里主要是判断方法级别的注解类型，并解析，进入parseHttpMethodAndPath("GET", ((GET) annotation).value(), false)方法

~~~java
   private void parseHttpMethodAndPath(String httpMethod, String value, boolean hasBody) {
      if (this.httpMethod != null) {
        throw methodError("Only one HTTP method is allowed. Found: %s and %s.",
            this.httpMethod, httpMethod);
      }
      this.httpMethod = httpMethod;
      this.hasBody = hasBody;

      if (value.isEmpty()) {
        return;
      }

      // Get the relative URL path and existing query string, if present.
      int question = value.indexOf('?');
      if (question != -1 && question < value.length() - 1) {
        // Ensure the query string does not have any named parameters.
        String queryParams = value.substring(question + 1);
        Matcher queryParamMatcher = PARAM_URL_REGEX.matcher(queryParams);
        if (queryParamMatcher.find()) {
          throw methodError("URL query string \"%s\" must not have replace block. "
              + "For dynamic query parameters use @Query.", queryParams);
        }
      }

      this.relativeUrl = value;
       //解析path中信息，如@GET（"/user/{username}"）中username的信息
      this.relativeUrlParamNames = parsePathParameters(value);
    }

~~~

这里主要为了解决诸如@GET（“/path?username={username}”）的问题，因为？后面不能追加{}，要用@Query来替代，下面到了解析方法中参数的注解了

~~~java
parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
~~~

可知serviceMethod会把解析下来的参数注解信息保存下来

~~~java
    private ParameterHandler<?> parseParameter(
        int p, Type parameterType, Annotation[] annotations) {
      ParameterHandler<?> result = null;
      for (Annotation annotation : annotations) {
        ParameterHandler<?> annotationAction = parseParameterAnnotation(
            p, parameterType, annotations, annotation);

        if (annotationAction == null) {
          continue;
        }

        if (result != null) {
          throw parameterError(p, "Multiple Retrofit annotations found, only one allowed.");
        }

        result = annotationAction;
      }

      if (result == null) {
        throw parameterError(p, "No Retrofit annotation found.");
      }

      return result;
    }
~~~

进入parseParameterAnnotation方法

~~~java
  private ParameterHandler<?> parseParameterAnnotation(
        int p, Type type, Annotation[] annotations, Annotation annotation) {
      if (annotation instanceof Url) {
      ......
      } else if (annotation instanceof Path) {
        if (gotQuery) {
          throw parameterError(p, "A @Path parameter must not come after a @Query.");
        }
        if (gotUrl) {
          throw parameterError(p, "@Path parameters may not be used with @Url.");
        }
        if (relativeUrl == null) {
          throw parameterError(p, "@Path can only be used with relative url on @%s", httpMethod);
        }
        gotPath = true;

        Path path = (Path) annotation;
        String name = path.value();
        validatePathName(p, name);
          //从retrofit中寻找能够把参数的实际类型转换成string的conveter
        Converter<?, String> converter = retrofit.stringConverter(type, annotations);
          //返回ParameterHandler的子类Path
        return new ParameterHandler.Path<>(name, converter, path.encoded());

      } else if (annotation instanceof Query) {
       ......
          Converter<?, String> converter =
              retrofit.stringConverter(iterableType, annotations);
          return new ParameterHandler.Query<>(name, converter, encoded).iterable();
        } else if (rawParameterType.isArray()) {
          Class<?> arrayComponentType = boxIfPrimitive(rawParameterType.getComponentType());
          Converter<?, String> converter =
              retrofit.stringConverter(arrayComponentType, annotations);
          return new ParameterHandler.Query<>(name, converter, encoded).array();
        } else {
          Converter<?, String> converter =
              retrofit.stringConverter(type, annotations);
          return new ParameterHandler.Query<>(name, converter, encoded);
        }

      } else if (annotation instanceof QueryName) {
          ......
      }
      ......
      return null; // Not a Retrofit annotation.
    }
~~~

参数注解的解析有三步

1. 判断注解类型

2. 获取converter并转换参数成string，比如把一个User对象转换成json

3. 把任务交给ParamterHandler的子类

<center style="color=black;font-size:120%">小结</center>
最后还是贴张图
![](image\Retrofit.png)



<p align="right" style="color:black"> 不正当之处请见谅</p>


