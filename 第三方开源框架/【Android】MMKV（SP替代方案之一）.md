# 【Android】MMKV—高性能轻量化存储组件

> 本文参考以及学习文档：
>
> [Android存储：轻松掌握MMKV通过学习本文，轻松掌握腾讯开发的 MMKV 组件，尽早在项目中替换掉SharedPr - 掘金](https://juejin.cn/post/7280436457136226343)
>
> [MMKV——Android上的使用(替换SP存储)MMKV 是基于 mmap 内存映射的 key-value 组件，底层 - 掘金](https://juejin.cn/post/6985530735973105701)
>
> [Android：MMKV 组件入门 - 简书](https://www.jianshu.com/p/e2e7373d3b50)
>
> [MMKV/README_CN.md at master · Tencent/MMKV](https://github.com/Tencent/MMKV/blob/master/README_CN.md)
>
> [资深Android研发大佬详解MMKV：谷歌都推荐的轻量级存储方案 - 知乎](https://zhuanlan.zhihu.com/p/362743850)

## MMKV是什么

在官方文档里

> MMKV 是基于 mmap 内存映射的 key-value 组件，底层序列化/反序列化使用 protobuf 实现，性能高，稳定性强。从 2015 年中至今在微信上使用，其性能和稳定性经过了时间的验证。近期也已移植到 Android / macOS / Windows / POSIX / HarmonyOS NEXT 等平台，一并开源。

简单来说

它就是一个【可以用于替代SP】，操作与SP类似的存储组件。

## 为什么要使用MMKV

**1. 读写效率低**

主要原因是其本身的读写方式导致的：

- 读写方式：I/O
- 数据格式：xml
- 写入方式：全量更新

即每当需要更新一项数据，SharedPreferences的整个读写过程都是：将**「所有数据」**转化成xml格式 -> 通过I/O方式写入。



**2. 容易导致ANR**

主要是由于同步提交(commit)、异步提交(Apply) 和 获取数据getXX()导致的。

```java
/*
 * 1. 同步提交commit
 * commit提交是同步的，直到磁盘操作成功后才会完成
 * 所以当数据量比较大时，使用commit很可能引起ANR
 */
  public boolean commit() {
    MemoryCommitResult mcr = commitToMemory();
    SharedPreferencesImpl.this.enqueueDiskWrite(mcr, null);
    try {
        mcr.writtenToDiskLatch.await();
    } catch (InterruptedException e) {
        return false;
  }

  /*
   * 回调的时机：
   * 1. commit是在内存和硬盘操作均结束时回调
   * 2. apply是内存操作结束时就进行回调
   */
   notifyListeners(mcr);
   return mcr.writeToDiskResult;

}

/*
 * 2. 异步提交apply
 * 当数据量比较大时，使用apply也可能引起ANR
 */
  public void apply() {
      final long startTime = System.currentTimeMillis();

      final MemoryCommitResult mcr = commitToMemory();
      final Runnable awaitCommit = new Runnable() {
              @Override
              public void run() {

                  // 启用等待
                  mcr.writtenToDiskLatch.await(); 
                  ......
              }
          };

      // 将 awaitCommit 添加到队列 QueuedWork 中
      QueuedWork.addFinisher(awaitCommit);

      Runnable postWriteRunnable = new Runnable() {
              @Override
              public void run() {
                  awaitCommit.run();
                  QueuedWork.removeFinisher(awaitCommit);
              }
          };
      // 将写入任务加入到队列中，而写入任务在一个线程中执行
      SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);

      // 为了保证异步任务及时完成，当生命周期处于 handleStopService() 、handlePauseActivity() 、handleStopActivity()时会调用QueuedWork.waitToFinish() 会等待写入任务执行完毕
      // waitToFinish() ：会一直等待写入任务执行完毕，其它什么都不做。
      // 当有很多写入任务，会依次执行；当文件很大时，效率很低，则容易造成 ANR
      public static void waitToFinish() {
      Runnable toFinish;
      while ((toFinish = sPendingWorkFinishers.poll()) != null) {
          toFinish.run(); 
      }

/*
 * 3. 获取数据getXX()
 * 所有 getXXX() 方法都是同步的，在主线程调用 get 方法，必须等待 SP 加载完毕，也有可能导致ANR
 */

  // 使用getSharedPreferences() 最终会调用SharedPreferencesImpl#startLoadFromDisk()开启一个线程异步读取数据
  new Thread("SharedPreferencesImpl-load") {
      public void run() {
          loadFromDisk();
      }
  }.start();

  // 当我们正在读取一个比较大的数据，还没读取完，接着调用 getXXX()。
  public String getString(String key, @Nullable String defValue) {
    synchronized (mLock) {
        awaitLoadedLocked();
        String v = (String)mMap.get(key);
        return v != null ? v : defValue;
    }
}
  // 在同步方法内调用了wait()，会一直等待 getSharedPreferences() 开启的线程读取完数据才能继续往下执行
  // 如果读取一个大的文件，也很大可能会造成ANR
  private void awaitLoadedLocked() {
      while (!mLoaded) {
          try {
              mLock.wait();
          } catch (InterruptedException unused) {
          }
      }
  }
}
```

## MMKV的使用

### 依赖引入

首先引入依赖：

```java
dependencies {
    implementation 'com.tencent:mmkv-static:1.2.10'
}
```

### 初始化以及修改根目录

```java
   MMKV.initialize(this)//(与下面的几选一，一般就使用这个就行)
   
   String dir = getFilesDir().getAbsolutePath() + "/mmkv";
   String rootDir = MMKV.initialize(dir);
```

### 获取MMKV实例

```java
import com.tencent.mmkv.MMKV;
//……

//1. 获取默认全局实例 (一般就使用这个就行)
MMKV kv = MMKV.defaultMMKV();

//2. 也可以自定义MMKV对象，设置自定ID  (根据业务区分的存取实例)
MMKV kv = MMKV.mmkvWithID("ID");

//3. MMKV默认是支持单进程的，如果业务需要多进程访问，需要在初始化的时候添加多进程模式参数
MMKV kv = MMKV.mmkvWithID("ID", MMKV.MULTI_PROCESS_MODE); //多进程同步支持
```

### 存取数据

```java
/** 添加/更新数据 **/
//存boolean类型
kv.encode("bool", true);
//存int类型
kv.encode("int", Integer.MIN_VALUE);
//存string类型
kv.encode("string", "MyiSMMKV");


/** 获取数据 **/
//获取boolean类型数据
boolean bValue = kv.decodeBool("bool");
//获取int类型数据
int iValue = kv.decodeInt("int");
//获取string类型数据
String str = kv.decodeString("string");
//...等类型的获取

// 删除数据
mmkv.removeValueForKey(key);
```



## MMKV原理

