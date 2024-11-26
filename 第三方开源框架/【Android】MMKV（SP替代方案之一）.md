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

### SharedPreference缺陷

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

### MMKV性能优势

写入随机int 1k次。

![img](https://github.com/Tencent/MMKV/wiki/assets/profile_android_mini.png)

> 支持的数据类型
>
> 支持以下 Java 语言基础类型：
>
> - `boolean、int、long、float、double、byte[]`
>
> 支持以下 Java 类和容器：
>
> - `String、Set<String>`
> - 任何实现了`Parcelable`的类型

## MMKV的使用

### 依赖引入

首先引入依赖：

```java
dependencies {
    implementation 'com.tencent:mmkv:2.0.1'
    // replace "2.0.1" with any available version
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

### 内存准备

通过 mmap 内存映射文件，提供一段可供随时写入的内存块，App 只管往里面写数据，由操作系统负责将内存回写到文件，不必担心 crash 导致数据丢失。

> ## 什么是MMAP？
>
> `mmap` 是一种在 Linux 和其他类 Unix 操作系统中使用的系统调用，它允许程序将一个文件或者设备映射到内存中。这样做的好处是可以直接通过内存操作来访问文件内容，而不需要使用传统的 read 和 write 系统调用。`mmap` 提供了一种高效的方式来处理文件数据，特别是在需要频繁读写大文件的场景下。
>
> **内存映射**：`mmap` 将文件内容映射到进程的地址空间，使得文件内容看起来就像是内存中的一部分。
>
> **高效访问**：由于文件内容被映射到内存，所以访问文件数据就像访问内存一样快。
>
> **共享内存**：使用 `mmap` 可以创建共享内存区域，这对于进程间通信（IPC）非常有用。
>
> **自动同步**：对映射区域的修改会自动同步回文件，无需显式调用 write 操作。
>
> **内存管理**：`mmap` 可以帮助操作系统更好地管理内存，因为它允许操作系统在需要时将文件内容从物理内存中换出到磁盘。
>
> **文件锁定**：`mmap` 可以用于文件锁定，防止其他进程修改文件。
>
> **支持匿名映射**：除了文件映射外，`mmap` 还支持创建匿名映射，这种映射不与任何文件关联，通常用于动态内存分配。



> ## 啥是Crash
>
> 在Android开发中，"Crash"指的是应用程序在运行时遇到未处理的错误导致程序异常终止的现象，也就是我们常说的“崩溃”或“闪退”。Crash会严重影响用户体验，可能导致数据丢失、用户流失甚至声誉受损。
>
> Crash可以分为两大类：
>
> 1. **未捕获异常崩溃**：通常是由于代码中的错误引发，如空指针异常（NullPointerException）、数组越界（ArrayIndexOutOfBoundsException）等。
> 2. **资源问题引起的崩溃**：包括内存溢出、网络连接问题等，这类问题会触发Android的ANR（Application Not Responding）对话框。
>
> Crash和ANR的区别
>
> **Crash（崩溃）**：
>
> - **定义**：指的是应用程序在运行时遇到了无法处理的异常，导致应用程序意外终止。这通常是由于代码中存在错误，如空指针引用、数组越界、类型转换错误等。
> - **表现**：应用突然关闭，用户被强制返回到主屏幕或应用列表，可能会看到“应用程序无响应”的提示，但这个提示并不是ANR，而是Crash发生后系统给出的反馈。
> - **原因**：通常是代码逻辑错误或资源管理不当（如内存泄漏）。
>
> **ANR（应用程序无响应）**：
>
> - **定义**：指的是应用程序在一段时间内没有响应用户的输入事件（如按键、触摸等），系统会弹出一个对话框提示用户是否要关闭该应用。
> - **表现**：用户界面冻结，用户操作没有响应，经过一定时间后（通常是5秒），系统会弹出ANR对话框。
> - **原因**：通常是因为主线程（UI线程）被长时间占用，导致无法处理输入事件。常见的原因包括主线程中进行了耗时的I/O操作、复杂的计算、长时间的等待等。

### **数据组织**

数据序列化方面选用 protobuf 协议，pb 在性能和空间占用上都有不错的表现。

### **写入优化**

考虑到主要使用场景是频繁地进行写入更新，我们需要有增量更新的能力。我们考虑将增量 kv 对象序列化后，append 到内存末尾。

### **空间增长**

使用 append 实现增量更新带来了一个新的问题，就是不断 append 的话，文件大小会增长得不可控。我们需要在性能和空间上做个折中。
