---
layout:     post
title:      OkHttp3自定义请求Headers
subtitle:   
date:       2016-01-18
author:     lx8421bcd
header-img: img/post-bg-android.jpg
catalog: false
tags:
    - Android
    - Android开发随笔
---

[原文](https://blog.csdn.net/u011734444/article/details/50536411)由本人于2016年创作于CSDN，后续修改完善提交于此。 


近几天尝试使用Retrofit和OkHttp构建网络层，从官网配置了依赖链接后，惊奇的发现OkHttp3，Retrofit2，都与以前的版本不兼容，不仅包名不一样（OkHttp3.＊，以前的版本是com.squareup.okhttp.*）而且很多方法也被删掉了，目前只有Retrofit2在网上有少许资料，OkHttp3只能参考官方文档了。  

在构建网络层时会遇到一个问题就是要手动配置Http请求的Headers，写入缓存Cookie，自定义的User-Agent等参数，但是对于有几十个接口的网络层，我才不想用注解配置Headers，目前网上很多文章的方法真对这两个版本都不是很适用，有的给出的方法已经被删除，有的方法会报出异常 :(  

查阅OkHttp3的使用说明可知，可以使用Interceptor对Http请求进行拦截并编辑请求内容，不过OkHttp2和OkHttp3的使用方法不太一样。在网上查阅文章经常看到的```okHttpClient.interceptors().add(...)```方法在OkHttp3中已不适用，OkHttp3的```interceptors()```方法返回的是一个不可编辑的列表，尝试对其进行编辑会抛出```UnSupportedOperationException```，所以我们必须把添加interceptor这步放到初始化OkHttpClient来做。

这里直接贴出实现方法

```java
import okhttp3.Interceptor;
import okhttp3.OkHttpClient;
import okhttp3.Request;
import okhttp3.Response;
import retrofit2.Retrofit;

//你的网络管理类
public class AppNetworkManager {

    private ClientApi initRetrofitApiClient() {

        Retrofit retrofit = new Retrofit.Builder()
                .baseUrl(SERVER_URL)
                .client(generateOkHttpClient())
                .build();
        return retrofit.create(ClientAPI.class);
    }

    private OkHttpClient generateOkHttpClient() {
        OkHttpClient.Builder builder = new OkHttpClient.Builder();

        ......

        // 添加拦截器，编辑header
        builder.addInterceptor(new Interceptor() {
            @Override
            public Response intercept(Chain chain) throws IOException {
                Request request = chain.request()
                        .newBuilder()
                        .addHeader("Content-Type", "application/x-www-form-urlencoded; charset=UTF-8")
                        .addHeader("Accept-Encoding", "gzip, deflate")
                        .addHeader("Connection", "keep-alive")
                        .addHeader("Accept", "*/*")
                        .addHeader("Cookie", "add cookies here")
                        .build();
                return chain.proceed(request);
            }
        });

        ......

        return builder.build();
    }
}
```
这里需要注意的是OkHttp的Interceptor有两种，ApplicationInterceptor(应用拦截器)和NetoworkInterceptor(网络拦截器)，这里我们使用的是应用拦截器（appInterceptor()，添加网络拦截器使用addNetworkInterceptor()）。一般来说应用拦截器足以满足我们小幅度编辑Headers的需求，应用拦截器管的事比较少，所以比较方便，但是如果需要深度掌控Http请求过程，就要考虑使用网络拦截器了。