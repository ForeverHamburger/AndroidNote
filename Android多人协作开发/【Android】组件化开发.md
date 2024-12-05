# 【Android】组件化开发

## 组件分层

学习组件化之前，我们首先了解一下组件分层的概念。

![image-20241204133223315](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412041332466.png)



### 组件化开发问题点

> 1. 业务组件，如何实现单独运行调试？
> 2. 业务组件间 没有依赖，如何实现页面的跳转？
> 3. 业务组件间 没有依赖，如何实现组件间通信/方法调用？
> 4. 业务组件间 没有依赖，如何获取fragment实例？
> 5. 业务组件不能反向依赖壳工程，如何获取Application实例、如何获取Application onCreate()回调（用于任务初始化）？

