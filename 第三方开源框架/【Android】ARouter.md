# 【Android】ARouter一点都不全面解析

## 概述

ARouter 是阿里巴巴开源的 Android 组件化路由框架，旨在简化 Android 项目中的页面跳转、服务调用、数据传递等操作。ARouter 采用注解方式为开发者提供便捷的路由管理功能，通过路由路径来管理各个组件之间的通信，达到组件化解耦的效果。

- Activity、Fragment 的跳转
- 跳转带参数
- 自定义服务
- 自定义拦截器
- 拦截下沉
- URL 重定向

## 使用介绍

### 添加依赖

```java
android {
    defaultConfig {
        ...
        javaCompileOptions {
            annotationProcessorOptions {
                //arouter编译的时候需要的module名字
                arguments = [AROUTER_MODULE_NAME: project.getName()]
            }
        }
    }
}
 
dependencies { 
    ...
    implementation 'com.alibaba:arouter-api:1.5.1'
    annotationProcessor 'com.alibaba:arouter-compiler:1.5.1'
}
```

通常情况下，我们会将 `arouter-api` 依赖放在基础服务的模块中，因为所有模块都需要依赖这个库，而 `arouter-compiler` 依赖需要放在每个模块中。

> **`arouter-api`** 依赖放在基础模块中，所有模块都需要依赖该库。
>
> **`arouter-compiler`** 依赖放在每个需要路由的模块中，负责编译时生成路由代码。

### 初始化

Arouter 的初始化很简单，通常我们会在 `Application` 中进行初始化：

```java
if (isDebug()) {
    ARouter.openLog();    // 打印日志
    ARouter.openDebug();  // 开启调试模式（如果在 InstantRun 模式下运行，必须开启调试模式！线上版本需要关闭，否则有安全风险）
}
ARouter.init(mApplication);  // 尽可能早，推荐在 Application 中初始化
```

在这里，我们要特别注意：Arouter 的初始化最好放在 `Application` 中，而不是其他地方。虽然我们曾尝试将初始化放在欢迎页，但在进程被 kill 后重新回到当前 Activity 时，Arouter 的初始化标记为 `false`，因此最终还是将初始化放回 `Application` 中。

### 添加注解

#### @Route注解

`Route` 注解通常用于标记路由路径，可以用于 Activity、Fragment、Service 等组件。以下是几种常见的 `Route` 注解用法：

##### **为 Activity 添加路由**

```java
@Route(path = "/login/loginActivity")
public class LoginActivity extends AppCompatActivity {
    ...
}
```

##### **为全局序列化服务定义路由**

ARouter 提供了一个接口 `SerializationService`，该接口用于定义如何序列化和反序列化对象。我们可以实现该接口并通过 `@Route` 注解注册成一个服务，然后在其他模块中使用。

首先，我们需要实现 `SerializationService` 接口，并定义如何将对象转换为字符串，或将字符串转换为对象。

```java
@Route(path = "/yourservicegroupname/json")
public class JsonServiceImpl implements SerializationService {
    private Gson gson;

    @Override
    public <T> T json2Object(String input, Class<T> clazz) {
        return gson.fromJson(input, clazz);
    }

    @Override
    public void init(Context context) {
        gson = new Gson();
    }

    @Override
    public String object2Json(Object instance) {
        return gson.toJson(instance);
    }

    @Override
    public <T> T parseObject(String input, Type type) {
        return gson.fromJson(input, type);
    }
}
```

这里用了Route注解定义了SerializationService的序列化的方式，在使用withObject的时候会使用该SerializationService。

在定义完序列化服务之后，其他模块或组件在通过 ARouter 传递对象时，会自动使用该全局序列化服务进行序列化和反序列化操作。

在 ARouter 中使用 `withObject()` 方法来传递自定义对象时，ARouter 会自动查找并调用已注册的 `SerializationService` 来进行序列化处理。

```java
User user = new User("Jack", 25);
ARouter.getInstance().build("/test/activity")
    .withObject("user", user)
    .navigation();
```

在目标 `Activity` 或组件中接收该对象时，ARouter 会自动进行反序列化，并注入到目标组件中。

```java
@Route(path = "/test/activity")
public class TestActivity extends AppCompatActivity {

    @Autowired
    public User user;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ARouter.getInstance().inject(this);  // 注入参数

        // 使用接收到的 User 对象
        Log.d("TestActivity", "User: " + user.getName() + ", " + user.getAge());
    }
}
```

在目标 `Activity` 中，通过 `@Autowired` 注解接收传递的对象后，调用`ARouter.getInstance().inject(this)` 来注入参数。

> 怎么注册全局序列化服务？
>
> 其实使用@Route注解 并 实现 `SerializationService` 接口，就已经注册好了。

##### **定义全局降级策略**

全局降级策略用于在路由跳转失败时提供备用的处理方式。通常情况下，当我们使用 ARouter 进行跳转时，如果目标路由不存在或不可用，ARouter 会抛出异常或做一些默认的错误处理。但是，有时候我们希望在路由跳转失败时，提供一些额外的逻辑，比如记录日志、向用户展示错误提示，或者进行一些备选处理。这个时候，我们可以定义一个全局的降级策略。

在 ARouter 中，全局降级策略是通过实现 `DegradeService` 接口来定义的。实现该接口后，可以通过 `@Route` 注解将降级服务注册到 ARouter 中。

`DegradeService` 接口包含两个方法：

- **`onLost(Context context, Postcard postcard)`**：当目标路由不可用时调用，通常用于做降级处理，比如展示错误信息或者跳转到备用路径。
- **`init(Context context)`**：初始化方法，通常用于初始化一些服务或资源。

```java

@Route(path = "/yourservicegroupname/DegradeServiceImpl")
public class DegradeServiceImpl implements DegradeService {
    
    @Override
    public void onLost(Context context, Postcard postcard) {
        // 当目标路由无法找到时会调用该方法
        Log.d("DegradeService", "路由失败，路径：" + postcard.getPath());
        // 可以在这里进行一些降级操作，例如跳转到错误页面，或执行备用操作
    }

    @Override
    public void init(Context context) {
        // 初始化操作，通常是创建或配置一些资源
        Log.d("DegradeService", "DegradeService 初始化");
    }
}

```

##### **实现服务提供**

在 ARouter 中，**服务提供**（Service Provider）是通过实现接口的方式来提供具体的功能服务。服务可以是任何形式的业务逻辑组件，它们被 ARouter 管理，并且可以通过路由机制来访问。通过 ARouter 提供的服务机制，我们可以轻松地将一个服务注册到路由系统中，其他模块或组件只需要通过路由路径来调用该服务。

#### **实现服务接口**

ARouter 服务是通过实现特定的接口来提供服务的。`IProvider` 是所有 ARouter 服务的基接口。所有的服务提供者需要实现这个接口。

**`IProvider` 接口**：

> `init(Context context)`：初始化方法，用于进行资源的初始化，通常只会调用一次。
>
> `destroy()`：销毁方法，用于释放资源（可选）。

```java
public interface HelloService extends IProvider {
    String sayHello(String name);
}

@Route(path = "/common/hello", name = "测试服务")
public class HelloServiceImpl implements HelloService {
    @Override
    public String sayHello(String name) {
        Log.d("HelloServiceImpl", "hello, " + name);
        return "hello, " + name;
    }

    @Override
    public void init(Context context) {}
}
```

通过 `@Route` 注解，我们可以将服务类注册到 ARouter 路由表中。这样，其他模块就可以通过 ARouter 的路由路径来调用该服务。

调用服务时，可以通过 `ARouter.getInstance().build(path)` 来获取已经注册的服务，并通过 `navigation()` 方法调用。

```java
import com.alibaba.android.arouter.facade.Postcard;
import com.alibaba.android.arouter.launcher.ARouter;

// 调用服务
public void callHelloService() {
    // 通过 ARouter 获取服务实例
    HelloService helloService = (HelloService) ARouter.getInstance().build("/common/helloService").navigation();
    
    // 使用服务提供的功能
    if (helloService != null) {
        String message = helloService.sayHello("Tom");
        Log.d("ServiceCall", message);  // 输出: Hello, Tom
    }
}
```



#### `@Interceptor` 注解

这个注解可以说非常强大，它能拦截你的路由，什么时候让路由通过什么时候让路由不通过，完全靠该Interceptor注解可以控制。比如我有一个需求，在跳分享的时候，我想看有没有登录，如果没有登录做登录的操作，如果登了了才让分享。如果之前是不是在每一个路由的地方都得判断有没有登录，很繁琐，有了路由拦截器不用在跳转的地方判断。

```java
@Interceptor(priority = 8, name = "登录拦截")
public class LoginInterceptor implements IInterceptor {
    @Override
    public void process(Postcard postcard, InterceptorCallback callback) {
        String path = postcard.getPath();
        if ("/share/shareActivity".equals(path)) {
            String userInfo = DataSource.getInstance(ArouterApplication.application).getUserInfo();
            if (TextUtils.isEmpty(userInfo)) {
                callback.onInterrupt(new Throwable("还没有登录，去登录"));
            } else {
                callback.onContinue(postcard);
            }
        } else {
            callback.onContinue(postcard);
        }
    }

    @Override
    public void init(Context context) {
        Log.d("LoginInterceptor", "LoginInterceptor初始化了");
    }
}
```

拦截器可以定义优先级，如果有多个拦截器，会依次执行拦截器。

#### **预处理服务**

预处理服务意思是在路由navigation之前进行干扰路由，通过实现PretreatmentService接口，比如我想干扰在分享之前判断有没有登录，如果没有登录，自行判断逻辑：

```java
@Route(path = "/yourservicegroupname/pretreatmentService")
public class PretreatmentServiceImpl implements PretreatmentService {
    @Override
    public boolean onPretreatment(Context context, Postcard postcard) {
        if ("/share/ShareActivity".equals(postcard.getPath())) {
            String userInfo = DataSource.getInstance(ArouterApplication.application).getUserInfo();
            if (TextUtils.isEmpty(userInfo)) {
                Toast.makeText(ArouterApplication.application, "你还没有登录", Toast.LENGTH_SHORT).show();
                return false;  // 返回 false 表示自行处理跳转逻辑
            }
        }
        return true;  // 返回 true 表示继续跳转
    }

    @Override
    public void init(Context context) {}
}
```

其实在这个例子中我演示的拦截器功能和预处理服务功能是一样的，只不过预处理服务是早于拦截器的，等到分析源码的时候我们分析他们的具体区别。

#### **重定义URL跳转**

重定义 URL 的跳转路径，可以通过实现 `PathReplaceService` 接口。

```java
@Route(path = "/yourservicegroupname/pathReplaceService")
public class PathReplaceServiceImpl implements PathReplaceService {
    @Override
    public String forString(String path) {
        if ("/login/loginActivity".equals(path)) {
            return "/share/shareActivity";  // 重定向登录页到分享页
        }
        return path;
    }

    @Override
    public Uri forUri(Uri uri) {
        return null;  // 如果需要，可以重定向 Uri
    }

    @Override
    public void init(Context context) {}
}
```

上面我把登录的路由改成分享的路由，在实际项目中大家看看有什么适用的场景

### **发起路由**

在 ARouter 中，**发起路由**主要是通过调用 `ARouter.getInstance().build("/path/to/activity")` 来进行页面跳转或者服务调用。ARouter 的路由机制使得我们能够通过预定义的路径快速地导航到相应的组件，服务和功能。通过 ARouter，我们可以传递数据、控制跳转的回调等。

#### 基本路由跳转

```java
ARouter.getInstance().build("/test/activity").navigation();
```

`ARouter.getInstance()`：获取 ARouter 实例。

`build("/test/activity")`：指定目标路径，`/test/activity` 是目标页面或组件的路由路径。

`navigation()`：执行跳转操作。

#### 路由跳转并传递参数

```java
ARouter.getInstance().build("/test/1")
            .withLong("key1", 666L)
            .withString("key3", "888")
            .withObject("key4", new Test("Jack", "Rose"))
            .navigation()
```

`withLong("key1", 666L)`：传递一个 `long` 类型的参数，键名为 `"key1"`，值为 `666L`。

`withString("key3", "888")`：传递一个 `String` 类型的参数，键名为 `"key3"`，值为 `"888"`。

`withObject("key4", new Test("Jack", "Rose"))`：传递一个 `Object` 类型的参数，键名为 `"key4"`，值为一个 `Test` 类的对象。

上面能用withObject方法传object是因为在上面定义了JsonServiceImpl序列化方式的路由类。withObejct还可以传集合、map等：

```java
ARouter.getInstance().build("/share/shareActivity").withString("username", "zhangsan")
    .withObject("testBean", TestBean("lisi", 20))
    .withObject(
        "listBean",
        listOf<TestBean>(TestBean("wanger", 20), TestBean("xiaoming", 20))
    )
    .navigation()
```

这里注意了在路由目标类里面定义接收list、map的时候，接收对象的地方不能标注具体的实现类类型，应仅标注为list或map。

跳转写法: 跳转方法主要指navigation方法，其实说是跳转方法不太准确，因为它不仅仅是跳转用的，比如生成一个interceptor、service等都是通过navigation方法实现的。

navigation主要有下面几个方法：

**`onLost(Postcard postcard)`**

当路由找不到时，会调用这个方法。可能是由于路径错误、路由未注册等原因导致无法找到目标页面。我们可以在这里进行错误提示或其他操作。

```java
@Override
public void onLost(Postcard postcard) {
    // 路由找不到时执行
    Log.d("Router", "未找到路径: " + postcard.getPath());
}
```

**`onFound(Postcard postcard)`**

当路由成功找到目标组件时，会调用这个方法。此时可以进行一些日志记录、统计等操作。

```java
@Override
public void onFound(Postcard postcard) {
    // 成功找到目标路径
    Log.d("Router", "找到了路径: " + postcard.getPath());
}
```

**`onInterrupt(Postcard postcard)`**

当路由跳转被拦截时（例如通过拦截器进行的中断），会调用这个方法。通常情况下，我们会在拦截器中进行一些条件判断，例如检查用户是否已登录，如果没有登录就中断跳转。

```java
@Override
public void onInterrupt(Postcard postcard) {
    // 路由被中断时执行
    Log.d("Router", "路由被拦截: " + postcard.getPath());
}
```

**`onArrival(Postcard postcard)`**

当路由跳转成功到达目标组件时，会调用这个方法。通常我们会在这里执行一些跳转后的操作，比如更新界面、统计、初始化等。

```java
@Override
public void onArrival(Postcard postcard) {
    // 路由成功到达目标页面
    Log.d("Router", "成功跳转到: " + postcard.getPath());
}
```

## 结语

> 参考：
>
> [阿里ARouter全面全面全面解析(使用介绍+源码分析+设计思路)_arouter init-CSDN博客](https://blog.csdn.net/CallmeZhe/article/details/112851137)
>
> [【Android】ARouter的使用及源码解析-CSDN博客](https://blog.csdn.net/Patrick_yuxuan/article/details/144000024?spm=1001.2014.3001.5502)
