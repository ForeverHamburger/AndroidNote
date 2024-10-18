# 【Android】浅析OkHttp

`OkHttp` 是一个高效、轻量级的 HTTP 客户端库，主要用于 Android 和 Java 应用开发。它不仅支持同步和异步的 HTTP 请求，还支持许多高级功能，如连接池、透明的 GZIP 压缩、响应缓存、WebSocket 等。OkHttp3 是 OkHttp 的第三个大版本，在稳定性和功能性方面进行了很多改进。

今天主要介绍OkHttp3。

## OkHttp基本用法

### 使用前

配置gradle

```java
compile 'com.squareup.okhttp3：okhttp：3.2.0'
compile 'com.squareup.okio：okio：1.7.0'
```

### 异步 GET 请求

最简单的 GET 请求，请求博客地址，代码如下所示：

```java
	Request.Builder requestBuilder = new Request.Builder().url(＂http：//blog.csdn.net/itachi85＂)；
	requestBuilder.method(＂GET＂, null)；
	Request request = requestBuilder.build()；
	OkHttpClient mOkHttpClient = new OkHttpClient()；
	Call mcall = mOkHttpClient.newCall(request)；
	mcall.enqueue(new Callback() {
		@Override
		public void onFailure(Call call, IOException e) {

		}
		@Override
 		public void onResponse(Call call, Response response) throws IOException {
			String str = response.body().string()；
 			Log.d(TAG str)； 
 		}
 	})；
```

其基本步骤就是创建 OkHttpClient、Request 和 Call，最后调用 Call 的 enqueue()方法。但是 每次这么写很麻烦，肯定是要进行封装的。需要注意的是 onResponse 回调并非在 UI 线程。如 果想要调用同步 GET 请求，可以调用 Call 的 execute 方法。

### 同步 GET 请求

代码示例如下：

```java
OkHttpClient client = new OkHttpClient();

// 创建Request对象
Request request = new Request.Builder()
    .url("https://jsonplaceholder.typicode.com/posts")
    .build();

// 同步发送请求
try (Response response = client.newCall(request).execute()) {
    if (response.isSuccessful()) {
        // 打印响应内容
        System.out.println(response.body().string());
    } else {
        System.out.println("Request Failed");
    }
} catch (IOException e) {
    e.printStackTrace();
}
```

在同步请求中，`execute()` 方法会阻塞线程，直到服务器返回响应。这里 `Response` 通过 `try-with-resources` 自动关闭流。

### 异步 POST 请求

OkHttp 3 异步 POST 请求和 OkHttp 2.x 有一些差别，就是没有 FormEncodingBuilder 这个类， 替代它的是功能更加强大的 FormBody。这里访问淘宝 IP 库，代码如下所示：

```java
RequestBody formBody = new FormBody.Builder()
				.add(＂ip＂, ＂59.108.54.37＂)
				.build()；
Request request = new Request.Builder()
		.url(＂http：//ip.taobao.com/service/getIpInfo.php＂)
        .post(formBody)
		.build()；
OkHttpClient mOkHttpClient = new OkHttpClient()；
Call call = mOkHttpClient.newCall(request)；
call.enqueue(new Callback() {
	@Override
	public void onFailure(Call call, IOException e) {
	
	}
	@Override
	public void onResponse(Call call, Response response) throws IOException {
		String str = response.body().string()；
		Log.d(TAG, str)；
	}
})；
```

这与异步 GET 请求类似，只是多了用 FormBody 来封装请求的参数，并传递给 Request。

### 设置超时时间和缓存

和 OkHttp 2.x 有区别的是 OkHttp 3 不能通过 OkHttpClient 直接设置超时时间和缓存了，而 是通过 OkHttpClient.Builder 来设置。通过 builder 配置好 OkHttpClient 后用 builder.build()来返回 OkHttpClient。所以我们通常不会调用 new OkHttpClient()来得到 OkHttpClient，而是通过 builder.build()得到 OkHttpClient。另外，OkHttp 3 支持设置连接、写入和读取的超时时间，如下 所示：

```java
File sdcache = getExternalCacheDir()；
int cacheSize = 10 ＊ 1024 ＊ 1024；
OkHttpClient.Builder builder = new OkHttpClient.Builder()
				.connectTimeout(15, TimeUnit.SECONDS)
				.writeTimeout(20, TimeUnit.SECONDS)
				.readTimeout(20, TimeUnit.SECONDS)
				.cache(new Cache(sdcache.getAbsoluteFile(), cacheSize))；
mOkHttpClient = builder.build()；
```

### 取消请求

使用 call.cancel()可以立即停止一个正在执行的 call。当用户离开一个应用时或者跳到其他 界面时，使用 call.cancel()可以节约网络资源；另外，不管同步还是异步的 call 都可以取消，也 可以通过 tag 来同时取消多个请求。当构建一个请求时，使用 Request.Builder.tag(Object tag)来 分配一个标签，之后你就可以用 OkHttpClient.cancel(Object tag )来取消所有带有这个 tag 的 call。 具体代码如下所示：

```java
private ScheduledExecutorService executor = Executors.newScheduledThreadPool(1)；
private void cancel() {
    final Request request = new Request.Builder()
        .url(＂http：//www.baidu.com＂)
        .cacheControl(CacheControl.FORCE_NETWORK)//1
        .build()；
	Call call = null；
	call = mOkHttpClient.newCall(request)；
	final Call finalCall = call；
	//100ms 后取消 call
	executor.schedule(new Runnable() {
		@Override
		public void run() {
			finalCall.cancel()；
    	}
	}, 100, TimeUnit.MILLISECONDS)；
	call.enqueue(new Callback() {
		@Override
		public void onFailure(Call call, IOException e) {
    
        }
        @Override
        public void onResponse(Call call, Response response) throws IOException {
            if (null != response.cacheResponse()) {
                String str = response.cacheResponse().toString()；
                Log.d(TAG, ＂cache---＂ + str)；
            } else {
                String str = response.networkResponse().toString()；
                Log.d(TAG, ＂network---＂ + str)；
            }
        }
    })；
}
```

创建定时线程池，100 ms 后调用 call.cancel()来取消请求。为了能让请求耗时，在上面代码 注释 1 处设置每次请求都要请求网络，运行程序并且不断地调用 cancel 方法。Log 打印结果如 图 5-13 所示。

## 源码解析OkHttp



