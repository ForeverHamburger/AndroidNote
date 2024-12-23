# 【Android】ARouter源码

## 

话不多说直接开始

在使用ARouter的路由功能前，需要先初始化SDK，直接从init方法进去：

```java
ARouter.init(getApplication());
```

下面分析初始化ARouter SDK的源码如下：

```java
/**
 * ARouter门面
 */
public final class ARouter {
    private volatile static boolean hasInit = false;

    public static void init(Application application) {
        if (!hasInit) {
            // ARouter是门面模式,代码实现在_ARouter中,下面接着分析_ARouter
            hasInit = _ARouter.init(application);
            if (hasInit) {
                _ARouter.afterInit();
            }
        }
    }
}
```

下面接着分析_ARouter源码：

```java
final class _ARouter {
    private volatile static boolean hasInit = false;
    private volatile static ThreadPoolExecutor executor = DefaultPoolExecutor.getInstance();
    private static Handler mHandler;
    private static Context mContext;
    private static InterceptorService interceptorService;

    protected static synchronized boolean init(Application application) {
        mContext = application;
        // 主要初始化逻辑都在LogisticsCenter中
        LogisticsCenter.init(mContext, executor);
        hasInit = true;
        mHandler = new Handler(Looper.getMainLooper());
        return true;
    }

    static void afterInit() {
        interceptorService = (InterceptorService) ARouter.getInstance().build("/arouter/service/interceptor").navigation();
    }
}
```

LogisticsCenter.init()

```java
public class LogisticsCenter {
    private static Context mContext;
    static ThreadPoolExecutor executor;
    private static boolean registerByPlugin;

    public synchronized static void init(Context context, ThreadPoolExecutor tpe) throws HandlerException {
        mContext = context;
        executor = tpe;
        //load by plugin first
        loadRouterMap();
        if (registerByPlugin) {
            logger.info(TAG, "Load router map by arouter-auto-register plugin.");
        } else {
            // 1.关键代码routeMap
            Set<String> routerMap;
            // It will rebuild router map every times when debuggable.
            // 2.debug模式或者PackageUtils判断本地路由为空或有新版本
            if (ARouter.debuggable() || PackageUtils.isNewVersion(context)) {
                // 3.获取ROUTE_ROOT_PAKCAGE(com.alibaba.android.arouter.routes)包名下的所有类
                // arouter-compiler根据注解自动生成的类都放在com.alibaba.android.arouter.routes包下
                routerMap = ClassUtils.getFileNameByPackageName(mContext, ROUTE_ROOT_PAKCAGE);
                // 4.建立routeMap后保存到sp中，下次直接从sp中读取StringSet;逻辑见else分支;
                if (!routerMap.isEmpty()) {
                    context.getSharedPreferences(AROUTER_SP_CACHE_KEY, Context.MODE_PRIVATE).edit().putStringSet(AROUTER_SP_KEY_MAP, routerMap).apply();
                }
                // 5.更新本地路由的版本号
                PackageUtils.updateVersion(context);    // Save new version name when router map update finishes.
            } else {
                routerMap = new HashSet<>(context.getSharedPreferences(AROUTER_SP_CACHE_KEY, Context.MODE_PRIVATE).getStringSet(AROUTER_SP_KEY_MAP, new HashSet<String>()));
            }
            // 6.获取routeMap后,根据路由类型注册到对应的分组里
            for (String className : routerMap) {
                if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_ROOT)) {
                    // 7.加载root，类名以SUFFIX_ROOT(com.alibaba.android.arouter.routes.ARouter$$Root)开头
                    // 以<String,Class>添加到HashMap(Warehouse.groupsIndex)中
                    ((IRouteRoot) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.groupsIndex);
                } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_INTERCEPTORS)) {
                    // 8.加载interceptorMeta，类名以SUFFIX_INTERCEPTORS(com.alibaba.android.arouter.routes.ARouter$$Interceptors)开头
                    // 以<String,IInterceptorGroup>添加到UniqueKeyTreeMap(Warehouse.interceptorsIndex)中;以树形结构实现顺序拦截
                    ((IInterceptorGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.interceptorsIndex);
                } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_PROVIDERS)) {
                    // 9.加载providerIndex，类名以SUFFIX_PROVIDERS(com.alibaba.android.arouter.routes.ARouter$$Providers)开头
                    // 以<String,IProviderGroup>添加到HashMap(Warehouse.providersIndex)中
                    ((IProviderGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.providersIndex);
                }
            }
        }
    }
}

```

Warehouse

```java
class Warehouse {
    // Cache route and metas
    static Map<String, Class<? extends IRouteGroup>> groupsIndex = new HashMap<>();
    static Map<String, RouteMeta> routes = new HashMap<>();

    // Cache provider
    static Map<Class, IProvider> providers = new HashMap<>();
    static Map<String, RouteMeta> providersIndex = new HashMap<>();

    // Cache interceptor
    static Map<Integer, Class<? extends IInterceptor>> interceptorsIndex = new UniqueKeyTreeMap<>("More than one interceptors use same priority [%s]");
    static List<IInterceptor> interceptors = new ArrayList<>();
}
```

结合3.1小节自动生成的代码来分析，Warehouse.groupsIndex中存放的key就是@Route(path = “/second/activity”)注解中所指定的path，value就是class：

```java
public class ARouter$$Group$$second implements IRouteGroup {
  @Override
  public void loadInto(Map<String, RouteMeta> atlas) {
    atlas.put("/second/activity", RouteMeta.build(RouteType.ACTIVITY, SecondActivity.class, "/second/activity", "second", null, -1, -2147483648));
  }
}

public class ARouter$$Providers$$second_library implements IProviderGroup {
  @Override
  public void loadInto(Map<String, RouteMeta> providers) {
  }
}
```

### 发起路由操作

```java
ARouter.getInstance().build("/first/activity").navigation();
```

接下来就从入口处分析发起路由的源码实现：

```java
public final class ARouter {
    // 单例模式
    public static ARouter getInstance() {
        if (!hasInit) {
            throw new InitException("ARouter::Init::Invoke init(context) first!");
        } else {
            if (instance == null) {
                synchronized (ARouter.class) {
                    if (instance == null) {
                        instance = new ARouter();
                    }
                }
            }
            return instance;
        }
    }

    public Postcard build(String path) {
        // _ARouter也是单例模式
        return _ARouter.getInstance().build(path);
    }
}
```

同3.2小节，ARouter是门面模式，相应地ARouter.getInstance()的单例模式的实际调用也是在
_ARouter.getInstance()中， build(String path)、navigation()等代码实际实现都在_ARouter中，后面不再单独说明。

ARouter.build()

继续分析_ARouter.getInstance().build()方法，方法返回Postcard对象，该对象表示一次路由操作所需的全部信息：

```java
final class _ARouter {
    protected Postcard build(String path) {
        // 1.首先获取PathReplaceService，判断是否重写跳转URL，默认为空
        // 进阶用法可以自定义类实现PathReplaceService来实现重写跳转URL，见github README
        PathReplaceService pService = ARouter.getInstance().navigation(PathReplaceService.class);
        if (null != pService) {
            path = pService.forString(path);
        }
        // 2.构造Postcard对象
        return build(path, extractGroup(path), true);
    }
    
    /**
    * 取出path中的组路径: /后的第一个即group
    */
    private String extractGroup(String path) {
        String defaultGroup = path.substring(1, path.indexOf("/", 1));
        return defaultGroup;
    }

    /**
    * 构造Postcard对象
    */
    protected Postcard build(String path, String group, Boolean afterReplace) {
        // 1.同build(String path)中的说明，判断是否重写跳转URL，默认没有重写的实现，afterReplace为true
        if (!afterReplace) {
            PathReplaceService pService = ARouter.getInstance().navigation(PathReplaceService.class);
            if (null != pService) {
                path = pService.forString(path);
            }
        }
        // 2.构造Postcard对象
        return new Postcard(path, group);
    }
}
```

Postcard

Postcard表示明信片，代表一次路由操作的所需信息，如下所示，信息比较多，我们暂时先只关注其父类RouteMeta的group和path属性：

```java
public final class Postcard extends RouteMeta {
    // Base
    private Uri uri;
    private Object tag;             // A tag prepare for some thing wrong. inner params, DO NOT USE!
    private Bundle mBundle;         // Data to transform
    private int flags = 0;         // Flags of route
    private int timeout = 300;      // Navigation timeout, TimeUnit.Second
    private IProvider provider;     // It will be set value, if this postcard was provider.
    private boolean greenChannel;
    private SerializationService serializationService;
    private Context context;        // May application or activity, check instance type before use it.
    private String action;

    // Animation
    private Bundle optionsCompat;    // The transition animation of activity
    private int enterAnim = -1;
    private int exitAnim = -1;
}

public class RouteMeta {
    private RouteType type;         // Type of route
    private Element rawType;        // Raw type of route
    private Class<?> destination;   // Destination
    private String path;            // Path of route  路径
    private String group;           // Group of route 组
    private int priority = -1;      // The smaller the number, the higher the priority
    private int extra;              // Extra data
    private Map<String, Integer> paramsType;  // Param type
    private String name;
    private Map<String, Autowired> injectConfig;  // Cache inject config.
}
```

Postcard.navigation()

ARouter.getInstance().build(“/first/activity”)返回Postcard对象，接下来继续分析Postcard.navigation()：

```java
public final class Postcard extends RouteMeta {
    public Object navigation() {
        return navigation(null);
    }

    public Object navigation(Context context, NavigationCallback callback) {
        // 实际实现在ARouter中
        return ARouter.getInstance().navigation(context, this, -1, callback);
    }
}
```

_ARouter.navigation()

接下来继续分析_ARouter：

```java
final class _ARouter {
    /**
     * 执行路由流程，主要工作包括：预处理、完善路由信息、拦截、继续执行路由流程
     */
    protected Object navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {
        // 1.自定义预处理PretreatmentService；没有自定义预处理或者预处理完成后继续向下传递
        PretreatmentService pretreatmentService = ARouter.getInstance().navigation(PretreatmentService.class);
        if (null != pretreatmentService && !pretreatmentService.onPretreatment(context, postcard)) {
            return null;
        }
        // 设置Application Context
        postcard.setContext(null == context ? mContext : context);
        try {
            // 2.LogisticsCenter完善路由信息；详见3.3.6分析
            // 在我们的例子中postcard现在只有path和group信息,LogisticsCenter会完善要打开的Activity类、routeType等路由信息
            LogisticsCenter.completion(postcard);
        } catch (NoRouteFoundException ex) {
            // LogisticsCenter根据path和group信息完善路由信息时如果未找到，则回调onLost
            if (null != callback) {
                callback.onLost(postcard);
            } else {
                DegradeService degradeService = ARouter.getInstance().navigation(DegradeService.class);
                if (null != degradeService) {
                    degradeService.onLost(context, postcard);
                }
            }
            return null;
        }
        if (null != callback) {
            callback.onFound(postcard);
        }
        // 3.Postcard是否是绿色通道，是则继续执行_navigation；
        // 不是则执行interceptorService判断是否有拦截流程，本次暂不分析拦截流程；
        if (!postcard.isGreenChannel()) {   // It must be run in async thread, maybe interceptor cost too mush time made ANR.
            interceptorService.doInterceptions(postcard, new InterceptorCallback() {
                @Override
                public void onContinue(Postcard postcard) {
                    _navigation(postcard, requestCode, callback);
                }
                
                @Override
                public void onInterrupt(Throwable exception) {
                    if (null != callback) {
                        callback.onInterrupt(postcard);
                    }
                }
            });
        } else {
            // 4.继续执行_navigation流程
            return _navigation(postcard, requestCode, callback);
        }
        return null;
    }

    /**
     * 根据完善的Postcard,执行对应的路由逻辑
     */
    private Object _navigation(final Postcard postcard, final int requestCode, final NavigationCallback callback) {
        final Context currentContext = postcard.getContext();
        // 1.根据不同的routeType执行不同逻辑;我们的例子中routeType是ACTIVITY
        switch (postcard.getType()) {
            case ACTIVITY:
                // 2.从Postcard取出信息构造Intent;我们的例子中postcard.getDestination()是要打开的Activity类
                final Intent intent = new Intent(currentContext, postcard.getDestination());
                intent.putExtras(postcard.getExtras());
                // Set flags.
                int flags = postcard.getFlags();
                if (0 != flags) {
                    intent.setFlags(flags);
                }
                if (!(currentContext instanceof Activity)) {
                    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                }
                String action = postcard.getAction();
                if (!TextUtils.isEmpty(action)) {
                    intent.setAction(action);
                }
                runInMainThread(new Runnable() {
                    @Override
                    public void run() {
                        // 3.启动Activity
                        startActivity(requestCode, currentContext, intent, postcard, callback);
                    }
                });
                break;
            case PROVIDER:
                return postcard.getProvider();
            case BOARDCAST:
            case CONTENT_PROVIDER:
            case FRAGMENT:
                Class<?> fragmentMeta = postcard.getDestination();
                try {
                    Object instance = fragmentMeta.getConstructor().newInstance();
                    if (instance instanceof Fragment) {
                        ((Fragment) instance).setArguments(postcard.getExtras());
                    } else if (instance instanceof android.support.v4.app.Fragment) {
                        ((android.support.v4.app.Fragment) instance).setArguments(postcard.getExtras());
                    }
                    return instance;
                } catch (Exception ex) {
                    logger.error(Consts.TAG, "Fetch fragment instance error, " + TextUtils.formatStackTrace(ex.getStackTrace()));
                }
            case METHOD:
            case SERVICE:
            default:
                return null;
        }
        return null;
    }
}
```

LogisticsCenter.completion()

补充分析上述流程中LogisticsCenter.completion()的主要工作：

```java
public class LogisticsCenter {
    public synchronized static void completion(Postcard postcard) {
        // 1.从Warehouse.routes中查找Postcard的path所对应的RouteMeta
        RouteMeta routeMeta = Warehouse.routes.get(postcard.getPath());
        if (null == routeMeta) {
            // routeMet为空，则从groupsIndex查找；没查找到则不存在，查找到则动态添加
            if (!Warehouse.groupsIndex.containsKey(postcard.getGroup())) {
                throw new NoRouteFoundException(TAG + "There is no route match the path [" + postcard.getPath() + "], in group [" + postcard.getGroup() + "]");
            } else {
                // Load route and cache it into memory, then delete from metas.
                addRouteGroupDynamic(postcard.getGroup(), null);
                completion(postcard);   // Reload
            }
        } else {
            // 2.从Warehouse.routes中查找到Postcard所对应的RouteMeta后，完善Postcard信息
            postcard.setDestination(routeMeta.getDestination());
            postcard.setType(routeMeta.getType());
            postcard.setPriority(routeMeta.getPriority());
            postcard.setExtra(routeMeta.getExtra());
            Uri rawUri = postcard.getUri();
            if (null != rawUri) {   // Try to set params into bundle.
                Map<String, String> resultMap = TextUtils.splitQueryParameters(rawUri);
                Map<String, Integer> paramsType = routeMeta.getParamsType();
                if (MapUtils.isNotEmpty(paramsType)) {
                    // Set value by its type, just for params which annotation by @Param
                    for (Map.Entry<String, Integer> params : paramsType.entrySet()) {
                        setValue(postcard,
                                params.getValue(),
                                params.getKey(),
                                resultMap.get(params.getKey()));
                    }
                    // Save params name which need auto inject.
                    postcard.getExtras().putStringArray(ARouter.AUTO_INJECT, paramsType.keySet().toArray(new String[]{}));
                }
                // Save raw uri
                postcard.withString(ARouter.RAW_URI, rawUri.toString());
            }
            switch (routeMeta.getType()) {
                case PROVIDER:  // if the route is provider, should find its instance
                    // Its provider, so it must implement IProvider
                    Class<? extends IProvider> providerMeta = (Class<? extends IProvider>) routeMeta.getDestination();
                    IProvider instance = Warehouse.providers.get(providerMeta);
                    if (null == instance) { // There's no instance of this provider
                        IProvider provider;
                        try {
                            provider = providerMeta.getConstructor().newInstance();
                            provider.init(mContext);
                            Warehouse.providers.put(providerMeta, provider);
                            instance = provider;
                        } catch (Exception e) {
                            logger.error(TAG, "Init provider failed!", e);
                            throw new HandlerException("Init provider failed!");
                        }
                    }
                    postcard.setProvider(instance);
                    postcard.greenChannel();    // Provider should skip all of interceptors
                    break;
                case FRAGMENT:
                    postcard.greenChannel();    // Fragment needn't interceptors
                default:
                    break;
            }
        }
    }
}
```

另外，LogisticsCenter是如何知道path="/first/activity"的Postcard在补全信息时，其对应的RouteType是Activity，对应的类是FirstActivity.class呢，看3.1小节中注解自动生成的代码，就可以看出来，APT处理过程中就会生成其对应信息，然后在3.2.3中LogisticsCenter.init()会将这些信息记录下来：

```java
/**
 * DO NOT EDIT THIS FILE!!! IT WAS GENERATED BY AROUTER. */
public class ARouter$$Group$$first implements IRouteGroup {
    @Override
    public void loadInto(Map<String, RouteMeta> atlas) {
        atlas.put("/first/activity", RouteMeta.build(RouteType.ACTIVITY, FirstActivity.class, 
                "/first/activity", "first", null, -1, -2147483648));
    }
}
```

