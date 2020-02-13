---
title: BridgeInterceptor分析
categories: [Android, okhttp]
tags: [okhttp]
catalog: true
date: 2018-12-13
---

之前分析过，OkHttp里面有个拦截器链，每个拦截器分工不同，他从上一个拦截器中得到请求信息，然后发给下一个拦截器进行请求，下一个拦截器会把请求结果再返回给当前拦截器，当前拦截器处理完之后再返回给上一个拦截器。

在OkHttp中内置了几个拦截器，本篇继续分析第二个内置拦截器BridgeInterceptor.

所有的拦截器都实现了Interceptor接口，核心的方法就是intercept()。

看源码：

    @Override 
    public Response intercept(Chain chain) throws IOException {
        //从链中得到RetryAndFollowUpInterceptor处理过的request.
        Request userRequest = chain.request();  
        //重新创建一个Request.Builder，用于补充http请求信息
        Request.Builder requestBuilder = userRequest.newBuilder(); 
        
        //开始 对http请求信息进行补充
        RequestBody body = userRequest.body();
        if (body != null) {
            MediaType contentType = body.contentType();
            if (contentType != null) {
                requestBuilder.header("Content-Type", contentType.toString());
            }
            
            long contentLength = body.contentLength();
            if (contentLength != -1) {
                requestBuilder.header("Content-Length", Long.toString(contentLength));
                requestBuilder.removeHeader("Transfer-Encoding");
            } else {
                requestBuilder.header("Transfer-Encoding", "chunked");
                requestBuilder.removeHeader("Content-Length");
            }
        }
        
        if (userRequest.header("Host") == null) {
            requestBuilder.header("Host", hostHeader(userRequest.url(), false));
        }
        
        if (userRequest.header("Connection") == null) {
            requestBuilder.header("Connection", "Keep-Alive");
        }
        
        // If we add an "Accept-Encoding: gzip" header field we're responsible for also decompressing
        // the transfer stream. 
        // Accept-Encoding 是客户端发给服务器,声明客户端支持的编码类型
        // 这里是判断是否http头中是否设置了Accept-Encoding，
        // 如果没有设置编码类型，那么这里处理方式是默认为gzip格式，是告诉服务器，客户端支持gzip，
        // 在得到请求响应信息之后，需要判断是否对响应信息的body进行gzip解压的
        // 如果设置了，那么就用设置的格式
        boolean transparentGzip = false;
        if (userRequest.header("Accept-Encoding") == null) {
            transparentGzip = true;
            requestBuilder.header("Accept-Encoding", "gzip");
        }
        
        //此处比较重要，对cookie的处理，下文会仔细分析，先mark一下
        List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
        if (!cookies.isEmpty()) {
          requestBuilder.header("Cookie", cookieHeader(cookies));
        }
        
        if (userRequest.header("User-Agent") == null) {
          requestBuilder.header("User-Agent", Version.userAgent());
        }
            
        //结束对http请求信息的补充
        
        Response networkResponse = chain.proceed(requestBuilder.build());//通过chain调用下一个拦截器进行请求，并且得到http 响应
        
        HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());//对cookie进行处理，下文会统一分析，先mark一下
        
        Response.Builder responseBuilder = networkResponse.newBuilder()
            .request(userRequest);//创建一个response.Builder，用于下面的代码对response的处理
        
        //此处判断是否需要对response进行gzip解压，transparentGzip是上面代码中对header中是否设置了gzip支持的
        if (transparentGzip
                && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))//服务器响应之后，是否对响应信息进行gzip进行压缩
                && HttpHeaders.hasBody(networkResponse)) {//判断这个响应是否有body,有些接口可能就返回一个header，具体可以参考http协议
            GzipSource responseBody = new GzipSource(networkResponse.body().source());
            Headers strippedHeaders = networkResponse.headers().newBuilder()
                .removeAll("Content-Encoding")
                .removeAll("Content-Length")
                .build();
            responseBuilder.headers(strippedHeaders);
            responseBuilder.body(new RealResponseBody(strippedHeaders, Okio.buffer(responseBody)));
        }
        
        return responseBuilder.build();//将响应信息返回给上一个interceptor
    }
      

#### cookie处理
对于cookie,OkHttp提供了一个CookieJar的接口，让我们可以自定义cookie的存取。
我们先来看一下CookieJar：

    public interface CookieJar {
        /** A cookie jar that never accepts any cookies. */
        CookieJar NO_COOKIES = new CookieJar() {
            @Override 
            public void saveFromResponse(HttpUrl url, List<Cookie> cookies) {
            }
            
            @Override 
            public List<Cookie> loadForRequest(HttpUrl url) {
              return Collections.emptyList();
            }
        };
        
        void saveFromResponse(HttpUrl url, List<Cookie> cookies);
        
        List<Cookie> loadForRequest(HttpUrl url);
    }

一个CookieJar的实例NO_COOKIES，没有对cookie进行任何处理
两个方法：
- saveFromResponse() 这个方法我们可以自己实现持久化的操作
- loadForRequest()   这个方法我们可以自己实现，根据url获取对应的cookie

上面分析intercept()方法的时候遗留了一个问题，cookie部分的处理，简化一下intercept()方法：

    @Override 
    public Response intercept(Chain chain) throws IOException {
        Request userRequest = chain.request();  //从链中得到RetryAndFollowUpInterceptor处理过的request.
        Request.Builder requestBuilder = userRequest.newBuilder(); //重新创建一个Request.Builder，用于补充http请求信息
        
        //开始 对http请求信息进行补充
        RequestBody body = userRequest.body();
        ......
        
        //这从cookieJar中获取cookie,并放到header中
        List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
        if (!cookies.isEmpty()) {
          requestBuilder.header("Cookie", cookieHeader(cookies));
        }
        
        ......
            
        //结束对http请求信息的补充
        
        Response networkResponse = chain.proceed(requestBuilder.build());//通过chain调用下一个拦截器进行请求，并且得到http 响应
        
        //这里会将response中带过来的cookie保存到cookieJar中
        HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());//对cookie进行处理，下文会统一分析，先mark一下
        
        ......
        
        return responseBuilder.build();//将响应信息返回给上一个interceptor
    }
    
    //这个方法就是从header中将cookie保存到cookiejar中
    public static void receiveHeaders(CookieJar cookieJar, HttpUrl url, Headers headers) {
        if (cookieJar == CookieJar.NO_COOKIES) return;
        
        List<Cookie> cookies = Cookie.parseAll(url, headers);
        if (cookies.isEmpty()) return;
        
        cookieJar.saveFromResponse(url, cookies);
    }
    
#### 总结
OkHttp中的BridgeInterceptoer做的事情有如下几点：
1. 补充http请求信息
2. 从cookiejar中将cookie添加到http 请求头中
3. 调用下一个intercepter处理http request，并且得到http response
4. 处理http response中的cookie,并保存到cookiejar中
5. 判断是否需要gzip解压，如果需要则解压
6. 将处理完的response返回给上一个interceptor

当然这篇文章还有一些地方没有详细讲解，因为涉及到http 协议相关的知识，此处就会分析了。


下一篇我们将分析CacheInterceptor，这一片会是非常核心的部分，需要我们把细节吃透，才能更好的使用OkHttp缓存机制。