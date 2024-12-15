# 【Android】ARouter一点都不全面解析

## 概述

Arouter 是 Android 实现组件化的路由框架，主要涉及到以下功能：

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

**为 Activity 添加路由**

```java
@Route(path = "/login/loginActivity")
public class LoginActivity extends AppCompatActivity {
    ...
}
```

**为全局序列化服务定义路由**

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

这里用了Route注解定义了SerializationService的序列化的方式，在使用withObject的时候会使用该SerializationService，后面会讲到该情况。

**定义全局降级策略**

```java
@Route(path = "/yourservicegroupname/DegradeServiceImpl")
public class DegradeServiceImpl implements DegradeService {
    @Override
    public void onLost(Context context, Postcard postcard) {
        Log.d("DegradeServiceImpl", "没有找到该路由地址: " + postcard.getPath());
    }

    @Override
    public void init(Context context) {}
}
```

这里用了Route注解定义了SerializationService的序列化的方式，在使用withObject的时候会使用该SerializationService，后面会讲到该情况。

**实现服务提供**

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

这个例子是官网的写法，意思是通过Route注解实现提供服务，那怎么实现接收服务呢，下面会在另外一种注解的时候讲到。

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

调用 `ARouter.getInstance().build("/path/to/activity")` 来进行路由跳转：

```java
ARouter.getInstance().build("/test/activity").navigation();
```

主要是通build方法生成postCard对象，最后调用postCard的navigation方法。

传值写法：

```java
ARouter.getInstance().build("/test/1")
            .withLong("key1", 666L)
            .withString("key3", "888")
            .withObject("key4", new Test("Jack", "Rose"))
            .navigation()
```

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

这里注意了在路由目标类里面定义接收list、map的时候，接收对象的地方不能标注具体的实现类类型，应仅标注为list或map，否则会影响序列化中类型的判断，其他类似情况需要同样处理其他几种序列化的方式也带了，大家自行查看 postCard 的 with 相关方法：

<img src="https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412152116018.png" alt="img" style="zoom:50%;" />

跳转写法: 跳转方法主要指navigation方法，其实说是跳转方法不太准确，因为它不仅仅是跳转用的，比如生成一个interceptor、service等都是通过navigation方法实现的，下一节介绍源码的时候会说到navigation有哪些具体作用。

![img](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412152117503.png)



navigation主要有下面几个方法，我们说下NavigationCallback对象，一看就是个回调：

```java
ARouter.getInstance().build("/share/shareActivity").withString("username", "zhangsan")
 
    .withObject("testBean", TestBean("lisi", 20))
    .withObject(
        "listBean",
        listOf<TestBean>(TestBean("wanger", 20), TestBean("xiaoming", 20))
    )
    .navigation(this, object : NavigationCallback {
        override fun onLost(postcard: Postcard?) {
        }
        override fun onFound(postcard: Postcard?) {
        }
        override fun onInterrupt(postcard: Postcard?) {
            Log.d("LoginActivity", "还没有登录")
        }
        override fun onArrival(postcard: Postcard?) {
        }
    })
```

实现了四个方法，onLost是找不到路由，onFound是找到路由，onInterrupt表示路由挂了，默认路由设置的超时时间是300s，onArrival表示路由跳转成功的回调，目前只在startActivity的回调，这个后面源码部分会讲到。

### **使用Gradle实现路由表自动加载**

可以说这个功能虽然是选项配置，但是对于arouter启动优化有很大的作用，我们项目在没使用这个gradle自动加载路由插件的时候初始化sdk需要4秒多，用了这个插件之后基本没消耗时间。

它主要是在编译期通过gradle插装把需要依赖arouter注解的类自动扫描到arouter的map管理器里面，在下一章我们通过反编译工具查看它是怎么插装代码的，而传统的是通过扫描dex文件来过滤arouter注解类来添加到map中。

**具体使用**

```java
//app的module的build.gradle
apply plugin: 'com.alibaba.arouter'
//工程的build.gradle
buildscript {
    repositories {
        jcenter()
    }
 
    dependencies {
        classpath "com.alibaba:arouter-register:1.0.2"
    }
}
```

