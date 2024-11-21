# 【Android】EventBus事件总线用法浅析

**EventBus** 是一个基于发布-订阅模式的 Android 库，专注于简化组件之间的通信。提供了一种高效、灵活且解耦的方式，用于在应用程序的不同组件（如 Activity、Fragment、Service 等）之间传递消息或事件。比如请求网络，两个Fragment之间需要通过Listener通信，这些需求都可以通过EventBus实现。

## **EventBus 的核心功能**

- **解耦组件：**
  发布者和订阅者之间不需要直接引用，降低了耦合性。
- **跨线程通信：**
  可以指定事件处理在哪个线程执行（主线程、后台线程等）。
- **事件优先级：**
  支持设置订阅者的优先级，以决定事件处理的先后顺序。

## EventBus使用方法

首先来看看EventBus的三要素！

> #### **1. Event（事件）**
>
> **定义**：事件是消息传递的载体，可以是任意类型的对象，例如自定义的类、基本数据类型的包装类等。
>
> **作用**：用来承载需要传递的数据。
>
> **使用方式**：开发者可以根据需求自定义事件类，携带需要传递的信息。

```java
public class MessageEvent {
    public final String message;

    public MessageEvent(String message) {
        this.message = message;
    }
}
```



> #### **Subscriber（订阅者）**
>
> **定义**：订阅者是接收事件的对象。在 EventBus 中，订阅者需要通过注解 `@Subscribe` 标记特定的方法来订阅事件。
>
> **注意**：EventBus 3.0 之前，订阅的方法名称必须是固定的（如 `onEvent`、`onEventMainThread` 等），而 3.0 之后方法名可以自定义，只需用 `@Subscribe` 注解即可。
>
> 订阅者需要指定线程模型，默认是 `POSTING` 模式。



> #### **Publisher（发布者）**
>
> **定义**：事件发布者是负责发送事件的组件，可以是应用中的任意部分。
>
> **作用**：将事件发送到事件总线，由总线分发给匹配的订阅者。
>
> **使用方式**：通过调用 `EventBus.getDefault().post(Object)` 发送事件。

```java
EventBus.getDefault().post(new MessageEvent("Hello EventBus!"));
```



再来看看 EventBus 的 4 种 ThreadMode 线程模型！

> POSTING(默认)：如果使用事件处理函数指定了线程模型为POSTING，那么该事件是在哪个线程发布出来的，事件处理函数就会在哪个线程中运行，也就是说发送事件和接收事件在同一个线程中。在线程模型为POSTING的事件处理函数中尽量避免执行耗时操作，因为它会阻塞事件的传递，甚至有可能会引起ANR(Application Not Responding)问题。
>
> MAIN：事件的处理会在切线程中执行。事件处理的时间不能太长，长了会引起ANR问题。
>
> BACKGROUND：如果事件是在U线程中发布出来的，那么该事件处理函数就会在新的线程中运行;如果事件本来就是在子线程中发布出来的，那么该事件处理函数直接在发送事件的线程中执行。在此事件处理函数中禁止进行UI更新操作。
>
> ASYNC：无论事件在哪个线程中发布，该事件处理函数都会在新建的子线程中执行，同样，此事件处理函数中禁止进行UI更新操作。



### EventBus 基本用法

EventBus 使用起来分为以下 5 个步骤：

（1）自定义一个事件类

```java
public class MessageEvent {
    ...
}
```

（2）在需要订阅事件的地方注册事件

```java
EventBus.getDefault().register(this);
```

（3）发送事件

```java
EventBus.getDefault().post(messageEvent);
```

（4）处理事件

```java
@Subscribe(threadMode = ThreadMode.MAIN)
public void XXX(MessageEvent messageEvent) {
    ...
}
```

（5）取消事件订阅

```java
EventBus.getDefault().unregister(this);
```



### EventBus 的黏性事件

**黏性事件（Sticky Event）** 是 EventBus 提供的一种特殊事件传递机制，即在事件发送之后，即使没有订阅该事件，稍后订阅者也能接收到这个事件。主要用于处理订阅发生在事件发布之后的场景。

**黏性事件的原理**

当使用 `postSticky()` 方法发布事件时，EventBus 会将该事件保存到内部的缓存中（StickyEvent Map）。当订阅者订阅时，会立即检查是否有符合条件的黏性事件，如果有则立即分发给订阅者。这种机制特别适合需要保存全局状态或后续动态注册的场景。

### **详细实现步骤**

#### **1. 定义事件类**

```java
public class MessageEvent {
    private String message;

    public MessageEvent(String message) {
        this.message = message;
    }

    public String getMessage() {
        return message;
    }
}
```

#### **2. 订阅黏性事件**

在 `MainActivity` 中定义一个处理黏性事件的方法，并用 `@Subscribe` 注解标记，同时指定 `sticky = true`：

```java
@Subscribe(threadMode = ThreadMode.POSTING, sticky = true)
public void onStickyEvent(MessageEvent messageEvent) {
    tv_message.setText(messageEvent.getMessage());
}
```

`threadMode = ThreadMode.POSTING`：订阅者运行在事件发布的线程。

`sticky = true`：表示该方法订阅的是黏性事件。

```java
@Override
protected void onStart() {
    super.onStart();
    EventBus.getDefault().register(this); // 注册订阅者
}

@Override
protected void onStop() {
    super.onStop();
    EventBus.getDefault().unregister(this); // 注销订阅者
}
```

#### **3. 发布黏性事件**

在 `SecondActivity` 中，通过 `postSticky()` 方法发布黏性事件：

```java
bt_subscription.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        EventBus.getDefault().postSticky(new MessageEvent("黏性事件"));
        finish(); // 返回到 MainActivity
    }
});
```

在 `MainActivity` 中不要立即注册订阅者（不点击“注册事件”按钮）。

跳转到 `SecondActivity` 并点击“发送黏性事件”按钮，这会发布一个黏性事件并返回到 `MainActivity`。

回到 `MainActivity`，点击“注册事件”按钮，这时订阅方法会接收到之前发送的黏性事件，更新界面内容。

## EventBus使用示例

你是西安邮电大学的一名正在学习安卓的带学生，此时你需要开发一个应用，其中包含如下内容：

1. **登录页面：** 用户输入账号和密码，点击登录。
2. **首页：** 显示用户的个性化内容，如推荐文章、好友动态等。
3. **个人中心：** 显示用户的昵称、头像和其他信息。

当用户成功登录后，首页和个人中心需要同时刷新数据以展示用户的最新状态。例如，首页需要显示用户的推荐内容，个人中心需要更新用户的昵称和头像。

------

#### **常规实现的问题**

1. **接口直接调用：** 登录成功后，直接调用首页和个人中心的刷新接口。这种方法需要登录页面直接依赖首页和个人中心的逻辑，导致强耦合。
2. **广播机制：** Android 的广播可以用来解决，但实现过程较复杂，还需要注册和过滤广播。

------

#### **用 EventBus 实现**

1. **解耦：** 登录页面不需要知道首页和个人中心的具体实现，只需发布一个登录成功的事件。
2. **异步处理：** 首页和个人中心可以根据需要选择在主线程或后台线程中处理事件。

------

### **实现步骤**

#### 1. 定义事件类

创建一个表示用户登录成功的事件：

```java
public class LoginSuccessEvent {
    private final String userId;

    public LoginSuccessEvent(String userId) {
        this.userId = userId;
    }

    public String getUserId() {
        return userId;
    }
}
```

#### 2. 发布事件

在登录成功的逻辑中，发布事件通知其他组件：

```java
// 假设登录成功后获取到用户ID
String userId = "12345";
EventBus.getDefault().post(new LoginSuccessEvent(userId));
```

#### 3. 订阅事件

在需要响应登录成功事件的页面（如首页和个人中心），订阅事件：

**首页代码：**

```java
@Subscribe(threadMode = ThreadMode.MAIN)
public void onLoginSuccess(LoginSuccessEvent event) {
    // 刷新推荐内容
    refreshContent(event.getUserId());
}

@Override
protected void onStart() {
    super.onStart();
    EventBus.getDefault().register(this);
}

@Override
protected void onStop() {
    super.onStop();
    EventBus.getDefault().unregister(this);
}
```

**个人中心代码：**

```java
@Subscribe(threadMode = ThreadMode.MAIN)
public void onLoginSuccess(LoginSuccessEvent event) {
    // 更新用户头像和昵称
    updateUserInfo(event.getUserId());
}

@Override
protected void onStart() {
    super.onStart();
    EventBus.getDefault().register(this);
}

@Override
protected void onStop() {
    super.onStop();
    EventBus.getDefault().unregister(this);
}
```

> **模块解耦：** 登录页面只负责发布事件，其他页面自行订阅并处理。
>
> **线程管理：** 首页和个人中心可以指定线程模式，轻松处理 UI 更新或后台操作。
>
> **扩展性强：** 以后如果有更多页面需要响应登录成功事件，只需订阅 `LoginSuccessEvent`，无需修改现有代码。



## 源码解析 EventBus

前面讲到了 EventBus 的用法，本节将讲解 EventBus 的源码。

### EventBus 的构造方法

当我们要使用 EventBus 时，首先会调用 EventBus.getDefault() 来获取 EventBus 实例。现在看看 getDefault 方法做了什么，如下所示：

```java
public static EventBus getDefault() {
    if (defaultInstance == null) {
        synchronized (EventBus.class) {
            if (defaultInstance == null) {
                defaultInstance = new EventBus();
            }
        }
    }
    return defaultInstance;
}
```

这是一个单例模式，采用了双重检查锁定（DCL）。

> **双重检查锁定（DCL，Double-Checked Locking）** 是一种常见的线程安全优化方案，通常用于实现单例模式或延迟初始化。它的核心目的是在保证线程安全的前提下，尽量减少加锁操作对性能的影响。
>
> 双重检查锁定的目的
>
> 提供一种 **懒加载** 的机制：资源在需要时才被初始化。
>
> 减少同步带来的性能开销：避免每次调用都进入同步块。
>
> 双重检查锁定的核心思想是：
>
> **外层检查**：如果对象已经初始化，则直接返回，不需要进入同步代码块。
>
> **内层检查**：如果对象未初始化，则加锁，确保只有一个线程能进行初始化操作。
>
> **同步块外的检查** 提高性能，因为大多数情况下对象已经被初始化。



接下来看看 EventBus 的构造方法：

```java
public EventBus() {
    this.DEFAULT_BUILDER;
}

private static final EventBusBuilder DEFAULT_BUILDER = new EventBusBuilder();
```

这里 DEFAULT_BUILDER 是默认的 EventBusBuilder，用来构造 EventBus。

```java
this.init(EventBusBuilder.builder());
EventBus(EventBusBuilder builder) {
    subscriptionsByEventType = new HashMap<>();
    typesBySubscriber = new HashMap<>();
    stickyEvents = new ConcurrentHashMap<>();
    mainThreadPoster = new HandlerPoster(this, Looper.getMainLooper(), 10);
    backgroundPoster = new BackgroundPoster(this);
    asyncPoster = new AsyncPoster(this);
    indexCount = 1;
    subscriberMethodFinder = new SubscriberMethodFinder(builder,
        builder.strictMethodVerification,
        builder.ignoreGeneratedIndex,
        builder.logSubscriberExceptions,
        builder.sendNoSubscriberEvent,
        builder.sendSubscriberExceptionEvent,
        builder.throwSubscriberException);
    eventInheritance = builder.eventInheritance;
}
```

我们可以通过构造一个 EventBusBuilder 来对 EventBus 进行配置，这里采用了建造者模式。

### 订阅者注册

获取 EventBus 后，便可以将订阅者注册到 EventBus 中。下面来看一下 register 方法：

```java
public void register(Object subscriber) {
    Class<?> subscriberClass = subscriber.getClass();
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
    synchronized (this) {
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
            subscribe(subscriber, subscriberMethod);
        }
    }
}
```

可以看出register 方法做了两件事:一件事是查找订阅者的订阅方法,另一件事是订阅者的注册。在 SubscriberMethod 类中，主要用来保存订阅方法的 Method 对象、线程模式、事件类型、优先级、是否是黏性事件等属性。

（1）查找订阅者的订阅方法

上面代码通过 `findSubscriberMethods` 方法，反射获取订阅者中所有被 `@Subscribe` 标注的方法：

```java
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
    List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass); // 1

    if (subscriberMethods != null) {
        return subscriberMethods;
    }

    if (ignoreGeneratedIndex) {
        subscriberMethods = findUsingReflection(subscriberClass);
    } else {
        subscriberMethods = findUsingInfo(subscriberClass); // 3
    }

    if (subscriberMethods.isEmpty()) {
        throw new EventBusException("Subscriber " + subscriberClass +
            " and its super classes have no public methods with the @Subscribe annotation");
    } else {
        METHOD_CACHE.put(subscriberClass, subscriberMethods); // 2
        return subscriberMethods;
    }
}
```

从缓存中查找是否有订阅方法的集合，如果找到了就立即返回。如果缓存中没有，则根据 `ignoreGeneratedIndex` 属性的值来选择采用何种方法查找订阅方法的集合。

`ignoreGeneratedIndex` 属性表示是否忽略注解器生成的 `MyEventBusIndex`。

`ignoreGeneratedIndex` 的默认值是 `false`，可以通过 `EventBusBuilder` 来设置它的值。

在注释 2 处找到订阅方法的集合后，放入缓存，以免下次继续查找。我们在项目中经常通过 EventBus 单例模式来获取默认的 EventBus 对象，也就是 `ignoreGeneratedIndex` 为 `false` 的情况。

> ### **什么是 Generated Index？**
>
> EventBus 提供了一种优化机制，通过 **预生成订阅方法索引** 来减少在运行时查找订阅方法所花费的时间和资源。
>
> **默认行为**：EventBus 使用 **反射** 动态查找类中的订阅方法（即标记了 `@Subscribe` 注解的方法）。
>
> **优化机制**：开发者可以生成一个静态的索引类，包含订阅者的所有订阅方法信息，避免运行时使用反射。
>
> 通过生成的索引类，EventBus 可以直接通过这个索引快速定位订阅方法，从而提升性能。

> 假设你在 EventBus 初始化时希望禁用索引：
>
> ```java
> EventBus eventBus = EventBus.builder()
>         .ignoreGeneratedIndex(true) // 忽略索引
>         .build();
> ```
>
> 或者启用索引（默认值）：
>
> ```java
> java复制代码EventBus eventBus = EventBus.builder()
>         .ignoreGeneratedIndex(false) // 使用索引
>         .addIndex(new MyEventBusIndex()) // 添加生成的索引类
>         .build();
> ```

这种情况调用了注释 3 处的 `findUsingInfo` 方法。

```java
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
    FindState findState = prepareFindState();
    findState.initForSubscriber(subscriberClass);
    while (findState.clazz != null) {
        findState.subscriberInfo = getSubscriberInfo(findState); // 1
        if (findState.subscriberInfo != null) {
            SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods(); // 2
            for (SubscriberMethod subscriberMethod : array) {
                if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                    findState.subscriberMethods.add(subscriberMethod);
                }
            }
        } else {
            findUsingReflectionInSingleClass(findState); // 3
        }
        findState.moveToSuperClass();
    }
    return getMethodsAndRelease(findState);
}

private void findUsingReflectionInSingleClass(FindState findState) {
    Method[] methods;
    try {
        methods = findState.clazz.getDeclaredMethods();
    } catch (Throwable th) {
        methods = findState.clazz.getMethods();
        findState.skipSuperClasses = true;
    }

    for (Method method : methods) {
        int modifiers = method.getModifiers();
        if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & Modifiers.STATIC) == 0) {
            // IGNORE) == 0
            Class<?>[] parameterTypes = method.getParameterTypes();
            if (parameterTypes.length == 1) {
                Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                if (subscribeAnnotation != null) {
                    Class<?> eventType = parameterTypes[0];
                    if (findState.checkAdd(method, eventType)) {
                        ThreadMode threadMode = subscribeAnnotation.threadMode();
                        findState.subscriberMethods.add(new SubscriberMethod(
                            method,
                            eventType,
                            threadMode,
                            subscribeAnnotation.priority(),
                            subscribeAnnotation.sticky()
                        ));
                    }
                }
            }
        }
    }
}
```

通过 `getSubscriberInfo` 方法来获取订阅者信息。在我们开始查找订阅方法的时候并没有忽略注解器为我们生成的索引 `MyEventBusIndex`。如果我们通过 `EventBusBuilder` 配置了 `MyEventBusIndex`，便会获取 `subscriberInfo`。

调用 `subscriberInfo` 的 `getSubscriberMethods` 方法便可得到订阅方法的相关信息。

如果没有配置 `MyEventBusIndex`，便会执行 `findUsingReflectionInSingleClass` 方法，将订阅方法保存到 `findState` 中。

通过反射来获取订阅者中所有的方法，并根据方法的类型、参数和注解来找到订阅方法。找到订阅方法后将订阅方法的相关信息保存到 `findState` 中。

（2）订阅者的注册过程

在查找完订阅者的订阅方法以后，便开始对所有的订阅方法进行注册。我们再看 register 方法中，调用了 subscribe 方法来对订阅方法进行注册，如下所示：

```java
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
    Class<?> eventType = subscriberMethod.eventType;
    //为当前订阅者创建一个新的 Subscription 对象，包含订阅者和对应的订阅方法。Subscription 代表了订阅者与事件类型之间的关系。
    Subscription newSubscription = new Subscription(subscriber, subscriberMethod); 
    //这是通过 eventType 从 subscriptionsByEventType（事件类型与订阅者列表的映射）中获取的现有订阅者列表。如果某事件类型已经有订阅者，这个列表将包含所有的订阅者。CopyOnWriteArrayList 是线程安全的。
    CopyOnWriteArrayList<Subscription> subscriptions =
        subscriptionsByEventType.get(eventType);
    //如果当前 eventType 没有订阅者，则创建一个新的 CopyOnWriteArrayList 并将其与事件类型映射；如果已经有订阅者列表，则检查新订阅者是否已经被注册。如果已经注册，则抛出异常，避免重复注册。
    if (subscriptions == null) {
        subscriptions = new CopyOnWriteArrayList<>();
        subscriptionsByEventType.put(eventType, subscriptions);
    } else {
        // 判断订阅者是否已经被注册
        if (subscriptions.contains(newSubscription)) {
            throw new EventBusException("Subscriber " + subscriber.getClass() +
                " already registered to event " + eventType);
        }
    }
    // 订阅者有一个优先级 (priority)，如果优先级较高，则会被插入到列表前面。这里遍历现有订阅者列表，找到正确的位置插入新订阅者，使得订阅者列表按照优先级从高到低排序。i == size 表示新订阅者的优先级最低，应该放在列表末尾。
    int size = subscriptions.size();
    for (int i = 0; i <= size; i++) {
        if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
            subscriptions.add(i, newSubscription);
            break;
        }
    }
    // 维护一个 typesBySubscriber 映射，记录每个订阅者订阅的事件类型。每当订阅者订阅一个新的事件类型时，将该事件类型添加到订阅者的事件类型列表中。
    List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
    if (subscribedEvents == null) {
        subscribedEvents = new ArrayList<>();
        typesBySubscriber.put(subscriber, subscribedEvents);
    }
    subscribedEvents.add(eventType);
    if (subscriberMethod.sticky) {
        // 黏性事件的处理
        Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
        for (Map.Entry<Class<?>, Object> entry : entries) {
            Class<?> candidateEventType = entry.getKey();
            if (eventType.isAssignableFrom(candidateEventType)) {
                Object stickyEvent = entry.getValue();
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }
    } else {
        Object stickyEvent = stickyEvents.get(eventType);
        if (stickyEvent != null) {
            checkPostStickyEventToSubscription(newSubscription, stickyEvent);
        }
    }
}
```

**黏性事件的处理**：

如果当前订阅者方法被标记为 `sticky`（黏性事件），则 EventBus 会检查当前事件类型是否有黏性事件与之匹配。黏性事件是指在事件发布后，仍然可以将事件发送给新的订阅者。

`stickyEvents` 是一个存储黏性事件的映射集合，符合 `eventType` 的事件将会被发送给新的订阅者。调用 `checkPostStickyEventToSubscription` 来将黏性事件发送给新的订阅者。

如果 `subscriberMethod` 没有标记为 `sticky`，则直接检查当前事件类型是否有已存在的黏性事件，如果有，仍然会将事件发送给新订阅者。

> 1. 将订阅者与其订阅的事件类型注册到 `subscriptionsByEventType` 中。
> 2. 按 `@Subscribe` 中的 `priority`（优先级）排序。
> 3. 处理黏性事件，检查该事件类型是否有未处理的黏性事件。

### 事件的发送

在获取 EventBus 对象以后，可以通过 post 方法来进行对事件的提交。post 方法的源码如下所示：

```java
public void post(Object event) {
    // PostingThreadState 保存有事件队列和线程状态信息
    PostingThreadState postingState = currentPostingThreadState.get();
    // 获取事件队列，并将当前事件插入事件队列
    List<Object> eventQueue = postingState.eventQueue;
    eventQueue.add(event);
    if (postingState.isPosting) {
        postingState.isMainThread = Looper.getMainLooper() == Looper.myLooper();
        postingState.isPosting = true;
        if (postingState.canceled) {
            throw new EventBusException("Internal error. Abort state was not reset");
        }
        try {
            // 处理队列中的所有事件
            while (!eventQueue.isEmpty()) {
                postSingleEvent(eventQueue.remove(0), postingState);
            }
        } finally {
            postingState.isPosting = false;
            postingState.isMainThread = false;
        }
    }
}
```

首先从 PostingThreadState 对象中取出事件队列，然后再将当前的事件插入事件队列。最后将队列中的事件依次交由 postSingleEvent 方法进行处理，并移除该事件。看看 postSingleEvent 方法里做了什么。

```java
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
    Class<?> eventClass = event.getClass();
    boolean subscriptionFound = false;
    // eventInheritance 表示是否向上查找事件的父类，默认为 true
    if (eventInheritance) {
        List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
        int countTypes = eventTypes.size();
        for (int h = 0; h < countTypes; h++) {
            Class<?> clazz = eventTypes.get(h);
            subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
        }
    } else {
        subscriptionFound |= postSingleEventForEventType(event, postingState, eventClass);
    }
    // 找不到该事件时的异常处理
    if (!subscriptionFound) {
        if (logSubscriberExceptions) {
            Log.d(TAG, "No subscribers registered for event " + eventClass);
        }
        if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class) {
            post(new NoSubscriberEvent(this, event));
        }
    }
}
```

eventinheritance 表示是否向上查找事件的父类，它的默认值为tue，可以通过 EventBusBuilder 进行配置。当 eventlnheritance为true 时，则通过 lookupAllEventTypes 找到所有的父类事件并存在 List 中,然后通过 postSingleEventForEventType 方法对事件逐一处理。

postSingleEventForEventType方法的源码如下所示:

```java
private boolean postSingleEventForEventType(Object event, PostingThreadState postingState,
    Class<?> eventClass) {
   CopyOnWriteArrayList<Subscription> subscriptions;
   synchronized (this) {
       subscriptions = subscriptionsByEventType.get(eventClass);
   }
   if (subscriptions != null && !subscriptions.isEmpty()) {
       for (Subscription subscription : subscriptions) {
           postingState.event = event;
           postingState.subscription = subscription;
           boolean aborted = false;
           try {
               postToSubscription(subscription, event, postingState);
               aborted = postingState.canceled;
           } finally {
               postingState.event = null;
               postingState.subscription = null;
               postingState.canceled = false;
           }
           if (aborted) {
               break;
           }
       }
       return true;
   }
   return false;
}
```

同步取出该事件对应的 Subscriptions（订阅对象集合）。遍历 Subscriptions，将事件 event 和对应的 Subscription（订阅对象）传递给 postingState 并调用 postToSubscription 方法对事件进行处理。接下来看看 postToSubscription 方法。

```java
private void postToSubscription(Subscription subscription, Object event,
    boolean isMainThread) {
    switch (subscription.subscriberMethod.threadMode) {
        case POSTING:
            invokeSubscriber(subscription, event);
            break;
        case MAIN:
            if (isMainThread) {
                invokeSubscriber(subscription, event);
            } else {
                mainThreadPoster.enqueue(subscription, event);
            }
            break;
        case BACKGROUND:
            if (!isMainThread) {
                backgroundPoster.enqueue(subscription, event);
            } else {
                invokeSubscriber(subscription, event);
            }
            break;
        case ASYNC:
            asyncPoster.enqueue(subscription, event);
            break;
        default:
            throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
    }
}
```

取出订阅方法的 threadMode(线程模式)，之后根据 threadMode 来分别处理。如果 threadMode 是 MAIN，那么，若是发送事件的线程是主线程，则通过反射直接运行订阅的方法；若其不是主线程，则需要通过 MainThreadPoster 将我们的订阅事件添加到主线线程队列中。mainThreadPoster 是一个内部类，继承自 Handler，确保在主线程中执行事件处理。

### 订阅者取消注册

取消注册则需要调用 unregister 方法，如下所示：

```java
public synchronized void unregister(Object subscriber) {
    List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
    if (subscribedTypes != null) {
        for (Class<?> eventType : subscribedTypes) {
            unsubscribeByEventType(subscriber, eventType);
        }
        typesBySubscriber.remove(subscriber);
    } else {
        Log.w(TAG, "Subscriber to unregister was not registered before: " +
            subscriber.getClass());
    }
}
```

我们在订阅者注册的过程中讲过 typesBySubscriber，它是一个 Map 集合。通过 subscriber 找到 subscribedTypes（事件类型集合）。将 subscriber 对应的 eventType 从 typesBySubscriber 中移除。最后遍历 subscribedTypes，并调用 unsubscribeByEventType 方法：

```java
private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
    List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions != null) {
        int size = subscriptions.size();
        for (int i = 0; i < size; i++) {
            Subscription subscription = subscriptions.get(i);
            if (subscription.subscriber == subscriber) {
                subscription.active = false;
                subscriptions.remove(i);
                i--;
                size--;
            }
        }
    }
}
```

在上面代码注释 1 处通过 eventType 来得到对应的 Subscriptions（订阅对象集合），并在 for 循环中判断如果 Subscription（订阅对象）的 subscriber（订阅者）属性等于传进来的 subscriber，则从 Subscriptions 中移除该 Subscription。