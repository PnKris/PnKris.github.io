---
layout: post
title: OkHttpClient分析
categories: [Android, okhttp]
tags: [okhttp]
catalog: true
date: 2018-12-13
---
### 前言
OKHttp在Android领域可以算是最流行的网络框架了，所以很基础，也很重要，里面也有很多值得我们去学习的地方。
本系列的文章将从一个简单的请求开始一步步分析okHttp内部是如何进行网络请求的。

### 简单的请求

```java

    OkHttpClient client = new OkHttpClient();
    Request request = new Request.Builder().get().url("").build();
    Call call = client.newCall(request);
    call.enqueue(new Callback() {
        @Override
        public void onFailure(Call call, IOException e) {
        }

        @Override
        public void onResponse(Call call, Response response) throws IOException {
        }
    });
```
okhttp的用法非常简单：
1. 创建一个okhttpClient
2. 构建一个request请求
3. 创建一个call，执行对象，我们所有的请求都是通过这个call内部进行执行的
4. 调用enqueue()发起一个异步网络请求，此时不需要我们再去创建子线程了，okhttp内部帮我们处理了。当然还有一个同步方法execute(),这个方法需要我们将其放在子线程中进行执行，否则会阻塞主线程，Android中会报在主线程中发起网络请求的异常。

### 创建okhttp客户端

创建okhttpClient一般有两种方式：
1. 直接new OkHttpClient()
2. 通过new OkHttpClient.Builder().build()来构建一个client

#### OkHttpClient构造方法分析

我们来看一下OkHttpClient()方法中是什么样的：

```java
public OkHttpClient() {
    this(new Builder());
}
```

```java

private OkHttpClient(Builder builder) {
    this.dispatcher = builder.dispatcher;
    .....
    this.connectTimeout = builder.connectTimeout;
    this.readTimeout = builder.readTimeout;
    this.writeTimeout = builder.writeTimeout;
  }
```

从上面代码我们不难看出，直接去new 一个OkHttpClient对象，内部其实也是通过了一个builder来创建的，在构造方法中对client进行赋值操作。

这里的Builder是个建造者，通过配置Builder来创建一个OkHttpClient对象。

#### Builder分析
Builder是OkHttpClient类中的一个静态内部类，看一下Builder:

```java
public static final class Builder {
    Dispatcher dispatcher;
    ......

    public Builder() {
      dispatcher = new Dispatcher();
      ......
    }

    Builder(OkHttpClient okHttpClient) {
      this.dispatcher = okHttpClient.dispatcher;
      ......
    }
    ......
    /**
     * Sets the dispatcher used to set policy and execute asynchronous requests. Must not be null.
     */
    public Builder dispatcher(Dispatcher dispatcher) {
      if (dispatcher == null) throw new IllegalArgumentException("dispatcher == null");
      this.dispatcher = dispatcher;
      return this;
    }
    ......

    public OkHttpClient build() {
      return new OkHttpClient(this);
    }
  }
}
```
代码非常多，这里简化了一下以便分析：

1. Builder()

这个方法用于对Builder中的参数做一个默认值的初始化。

2. Builder(OkHttpClient okHttpClient)

这个方法表示的是我们可以根据一个已经创建好的OkHttpClient对象来创建一个参数一样的建造者，用于我们调用build方法创建新的OkHttpClient.
使用场景：
```java
OkHttpClient newClient = new OkHttpClient.Builder(oldClient).build();
```
那么这个newClient和oldClient的参数设置都是一样的

3. Builder dispatcher(Dispatcher dispatcher)

在Builder类中存在大量的类似于这种方法，返回值都是Builder,其实这个方法的作用是对Builder对象进行赋值，然后再把Builder对象返回出去，继续调用其他的这种方法，这样就构成了链式调用的写法，我们平时写代码的时候也可这样做。
使用场景：
当前我们的构造参数很多的时候，我们可以使用Builder这种方式，通过Builder中的方法来设置参数，返回一个Builder，做成链式调用。

4. build()方法实现

```java
public OkHttpClient build() {
    return new OkHttpClient(this);
}
```
可以看到，这里直接new了一个OkHttpClient对象，将Builder对象作为参数传入到OkHttpClient中。

#### OkHttpClient成员变量分析

OkHttpClient中的成员变量非常多，我们挑几个重要的变量介绍一下，对其有个认识，方便后续更深入阅读源码做个准备。

1. Dispatcher dispatcher;

分发器：对Request进行分发处理的一个管理类，包含了一些队列，线程池之类，将请求分发到对应的队列中，并处理。

2. List<Interceptor> interceptors;

拦截器列表：存放自定义的拦截器和OkHttpClient内置的拦截器

3. List<Interceptor> networkInterceptors;

网络拦截器列表：一些自定义的拦截器，跟interceptors有点类似，只是执行的顺序上有些不一样，在分析拦截器链的时候会讲到这一块，只要记住这个东西也是存放自定义拦截器的

4. CookieJar cookieJar;

Cookie管理:这是个接口，当我们对cookie进行处理的时候，需要我们实现这个接口，并且自己实现存取的操作代码，默认的是一个空的实现，也就是方法里面什么都没写。这里会在BridgeInterceptor中介绍到。

5. Cache cache;

设置cache的保存路径，以及cache的文件夹的大小，这里的cache是针对http协议中的首部字段来处理cache的，如果我们需要自定义一些接口请求数据的保存什么的，可以自己定义一个拦截器对请求和响应结果进行存取操作。这一块会在cacheInterceptor的分析中介绍。

6. ConnectionPool connectionPool;

连接池：http是基于tcp的实现的，如果每次请求完都要去断开连接，下次请求再去建立连接一次，会非常耗费资源的，所以就设计了一个连接池的东西。
当有请求的时候，先去连接池中找一个没有断开的连接进行请求，如果没有再创建一个新的连接，用完以后，不断开，放回到连接池中。当然真正的实现考虑的会比较多，比如超时断开什么的，有点类似于线程池，后面讲到connectionInterceptor的时候会详细解释一下。

那么OkHttpClient我们就分析到这里，下一篇会分析Request相关的内容。
