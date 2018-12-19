---
layout:     post
title:      Retrofit2根据接口声明决定是否带Cookie请求
subtitle:   一种按接口注解声明编辑请求Header的方法
date:       2017-01-04
author:     lx8421bcd
header-img: img/post-bg-android.jpg
catalog: false
tags:
    - Android
    - Android开发随笔
---

[原文](https://blog.csdn.net/u011734444/article/details/53992960)由本人于2017年创作于CSDN，后续修改完善提交于此。

注：  
后来和后端商议过后，决定不用按接口来区分是否发送cookie，也就不需要使用这个方法来处理Cookie了，后来客户端都使用[PersistentCookieJar](https://github.com/franmontiel/PersistentCookieJar)进行cookie管理，不过本方法作为Retrofit的一种按接口注解声明编辑Header的方法仍有其使用价值。

原文：  
整理自最近的项目经验，Retrofit关于cookie缓存与使用的文章已经很多了，无论是使用CookieJar还是在OkHttpClient build的时候通过Interceptor设置header，但是我在项目中有针对接口决定是否在请求中带cookie的需求，比如获取应用首页数据，推荐商品之类的账户无关接口不需要带cookie，而账户相关接口需要带cookie。  

最简单的实现方案是将接口分为需要cookie和不需要cookie两组分别声明，但是这样就限定了接口的声明方式，对于已有的项目改起来会比较麻烦，另一方面降低了声明的自由度，限制了其它方式的声明。是一种比较挫的做法。  

对于这个问题，我首先想到的是能不能像Retrofit那样通过注解为接口声明属性，比如写个@UseCookie注解，被这个注解标识的接口在请求时将Cookie设置在Request的Header中。  
要实现这个需求需要2点：
1. 解析自定义注解
2. 在解析自定义注解的期间能够编辑请求的Headers

这就需要阅读Retrofit的源码了 [retrofit github](https://github.com/square/retrofit/)  

通过阅读Retrofit的源码可以知道Retrofit将声明的method转化成具体的Call主要是在ServiceMethod中进行的，如果不想动Retrofit的源码，而是用其对外提供的接口实现Annotation解析的功能，那就只有在ConverterFactory和CallAdapterFactory中动手脚了。

来看一下继承Converter.Factory后所能重写的方法：
```java
public class AConverterFactory extends Converter.Factory {
    @Override
    public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations, Retrofit retrofit) {
        return super.responseBodyConverter(type, annotations, retrofit);
    }
 
    @Override
    public Converter<?, RequestBody> requestBodyConverter(Type type, Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) {
        return super.requestBodyConverter(type, parameterAnnotations, methodAnnotations, retrofit);
    }
 
    @Override
    public Converter<?, String> stringConverter(Type type, Annotation[] annotations, Retrofit retrofit) {
        return super.stringConverter(type, annotations, retrofit);
    }
}
```
可以看到，真的是已经很接近了，这里已经可以拿到注解进行判断了，可惜的是，方法要求的返回值是RequestBody。没有办法编辑Headers。如果在项目中Session不是通过Cookie传输而是放在body里面，那么自定义注解在这里就可以用了。  

如果说一定要用Headers的Cookie，那么就得另寻它路。回想在Retrofit2中为请求设置统一Headers的方法是在OkHttpClient中添加Interceptor，在Interceptor中配置Header；而通过Retrofit的Headers注解可以为具体method添加指定Header。那么应该可以在@Headers注解中为某一接口添加一个Key，在Interceptor编辑Headers的时候，如果检测到这个Key，则断定这个请求需要在Header中添加Cookie。  

先声明一个常量Key
```java
public class NetConstants {
    public static final String ADD_COOKIE = "Add-Cookie";
    ......
}
```
然后在具体接口中通过@Headers注解标识
```java
public interface ClientApi {
 
    @Headers(NetConstants.ADD_COOKIE)
    @GET("adat/sk/{cityId}.html")
    Call<ResponseBody> getWeather(@Path("cityId") String cityId);
}
```
在配置Retrofit Builder的时候，在Interceptor中添加判断：
```java
Interceptor configInterceptor = new Interceptor() {
    @Override
    public Response intercept(Chain chain) throws IOException {
        Request.Builder builder = chain.request().newBuilder();
 
        // add custom headers......
 
       if (chain.request().headers().get(NetConstants.ADD_COOKIE) != null) { 
            builder.removeHeader(NetConstants.ADD_COOKIE);
            if (!TextUtils.isEmpty(SessionManager.getSession())) {
                builder.header("Cookie", SessionManager.getSession());
            }
        }
 
        Request request = builder.build();
        if (BuildConfig.DEBUG) {
            Log.d("TAG", "request url : " + request.url());
        }
        return chain.proceed(request);
    }
};
Interceptor responseInterceptor = new Interceptor() {
    @Override
    public Response intercept(Chain chain) throws IOException {
        Response response = chain.proceed(chain.request());
        //存入Session
        if (response.header("Set-Cookie") != null) {
            SessionManager.setSession(response.header("Set-Cookie"));
        }
        //刷新API调用时间
        SessionManager.setLastApiCallTime(System.currentTimeMillis());
 
        return response;
    }
};
okHttpClientBuilder.addInterceptor(configInterceptor);
okHttpClientBuilder.addInterceptor(responseInterceptor);
mRetrofitBuilder.client(okHttpClientBuilder.build());
```
这样就可以实现根据具体的接口来决定是否在请求时附带Cookie了。