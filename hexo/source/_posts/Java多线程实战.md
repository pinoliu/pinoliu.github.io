---
title: Java多线程实战
date: 2018-09-08 21:46:33
subtitle: java_multi_thread
tags:
- java
- thread
---

## 核心类

FutureTask可用于异步获取执行结果或取消执行任务的场景。

通过传入Runnable或者Callable的任务给FutureTask，直接调用其run方法或者放入线程池执行，之后可以在外部通过FutureTask的get方法异步获取执行结果，因此，FutureTask非常适合用于耗时的计算，主线程可以在完成自己的任务后，再去获取结果。另外，FutureTask还可以确保即使调用了多次run方法，它都只会执行一次Runnable或者Callable任务，或者通过cancel取消FutureTask的执行等。同时，FutureTask存在一个回调函数protected void done()，当任务结束时，该回调函数会被触发。只需重载该函数，即可实现在线程刚结束时就做一些事情。

ListenableFuture顾名思义就是可以监听的Future，它是对java原生Future的扩展增强。

我们知道Future表示一个异步计算任务，当任务完成时可以得到计算结果。如果我们希望一旦计算完成就拿到结果展示给用户或者做另外的计算，就必须使用另一个线程不断的查询计算状态。这样做，代码复杂，而且效率低下。使用 ListenableFuture帮我们检测Future是否完成了，如果完成就自动调用回调函数，这样可以减少并发程序的复杂度。(ListenableFuture的addListener方法 与 通过Futures的静态方法addCallback给ListenableFuture添加回调函数)

## 代码实践

### maven依赖

```xml
		<dependency>
			<groupId>com.google.guava</groupId>
			<artifactId>guava</artifactId>
		</dependency>
```

### 核心类：FutureTask

#### 主函数 FutureTaskMain

```java
/**
 * 多线程的基础使用篇 
 * 核心使用类：FutureTask、BlockingQueue
 * 
 * @author LXiao
 *
 */
public class FutureTaskMain {
  public static void main(String[] args) {
    // LinkedBlockingQueue<E>:理论上容量没有边界，可不指定容量
    // ArrayBlockingQueue<E>(int)：构造时需要指定容量，并可选择是否公平调用（等待时长、效率低，通常不公平）
    BlockingQueue<Runnable> queue = new LinkedBlockingQueue<Runnable>();

    ThreadFactoryBuilder threadFactoryBuilder = new ThreadFactoryBuilder();
    threadFactoryBuilder.setNameFormat("future-%d");

    // 明确线程池的关键状态
    // 阿里巴巴 Java开发手册：不推荐使用Executors，不利于开发明确线程池的运行规则，规避资源耗尽的风险。
    ThreadPoolExecutor threadPoolExecutor =
        new ThreadPoolExecutor(10, 12, 3, TimeUnit.SECONDS, queue, threadFactoryBuilder.build());

    // FutureTask将Callable装换成Future和Runnable
    // FutureTaskImpl重载了done接口，线程结束后会优先回调该接口，然后解除get方法的阻塞
    // 如果业务更复杂，可参考Guava的Futures.addCallback，里面有更细化的 onFailure 和 onSuccess 两个回调。
    FutureTask<Integer> futureTask = new FutureTaskImpl(() -> {
      System.out.println("futureTask ID = " + Thread.currentThread().getId());
      System.out.println("futureTask Name = " + Thread.currentThread().getName());
      System.out.println("futureTask sleep(3000), start time = " + System.currentTimeMillis());
      Thread.currentThread().sleep(3000);
      System.out.println("futureTask sleep(3000), end time = " + System.currentTimeMillis());
      return 0;
    });

    // futureTask 只会执行一次，保障了在高并发环境下确保任务只执行一次
    threadPoolExecutor.submit(futureTask);
    threadPoolExecutor.submit(futureTask);

    // 只是拒绝新增任务，在队列中的依旧会执行
    threadPoolExecutor.shutdown();

    // 场景一：整体业务需要等待线程的结果
    // 实现：通过 futureTask.get 自带的阻塞等待获取结果
    try {
      // 可设置超时
      Integer result = futureTask.get();
      System.out.println("futureTask.get(), result = " + result);
    } catch (InterruptedException e) {
      System.out.println("futureTask.get(), sInterruptedException = " + e);
    } catch (ExecutionException e) {
      System.out.println("futureTask.get(), ExecutionException = " + e);
    }

    // 执行效果：
    // FutureTaskImpl init.
    // futureTask ID = 10
    // futureTask Name = future-0
    // futureTask sleep(3000), start time = 1536406390273
    // futureTask sleep(3000), end time = 1536406393288
    // FutureTaskImpl done.
    // futureTask.get(), result = 0

    // 场景二：在子线程未获得结果时，有其它任务可以进行
    // 实现：通过 futureTask.isDone() 进行循环等待，并进行适当sleep，避免无效的反复执行
    while (!futureTask.isDone()) {
      System.out.println("!futureTask.isDone(), time = " + System.currentTimeMillis());
      try {
        // 适当sleep，避免无效的反复执行
        Thread.currentThread().sleep(1000);
      } catch (InterruptedException e) {
        System.out.println("sleep(1000), InterruptedException = " + e);
      }
    }

    try {
      // 可设置超时
      Integer result = futureTask.get();
      System.out.println("futureTask.get(), result = " + result);
    } catch (InterruptedException e) {
      System.out.println("futureTask.get(), sInterruptedException = " + e);
    } catch (ExecutionException e) {
      System.out.println("futureTask.get(), ExecutionException = " + e);
    }

    // 执行效果：
    // FutureTaskImpl init.
    // !futureTask.isDone(), time = 1536406330077
    // futureTask ID = 10
    // futureTask Name = future-0
    // futureTask sleep(3000), start time = 1536406330077
    // !futureTask.isDone(), time = 1536406331093
    // !futureTask.isDone(), time = 1536406332097
    // futureTask sleep(3000), end time = 1536406333084
    // FutureTaskImpl done.
    // futureTask.get(), result = 0
  }
}

```

####  实现类：FutureTaskImpl

```java
/**
 * FutureTask实现类
 * 核心：实现了 done() 的回调方法，会在 FutureTask.get() 之前执行
 * 
 * @author LXiao
 *
 */
public class FutureTaskImpl extends FutureTask<Integer> {

  public FutureTaskImpl(Callable<Integer> callable) {
    super(callable);
    System.out.println("FutureTaskImpl init.");

  }

  @Override
  protected void done() {
    super.done();
    System.out.println("FutureTaskImpl done.");
  }
}
```



### 核心类：ListenableFuture

#### 主函数：GuavaMain

```java
/**
 * 多线程的基础使用篇 
 * 核心使用类：ListeningExecutorService、ListenableFuture、Futures、BlockingQueue
 * 
 * @author LXiao
 *
 */
public class GuavaMain {

  public static void main(String[] args) {
    // LinkedBlockingQueue<E>:理论上容量没有边界，可不指定容量
    // ArrayBlockingQueue<E>(int)：构造时需要指定容量，并可选择是否公平调用（等待时长、效率低，通常不公平）
    BlockingQueue<Runnable> queue = new LinkedBlockingQueue<Runnable>();

    ThreadFactoryBuilder threadFactoryBuilder = new ThreadFactoryBuilder();
    threadFactoryBuilder.setNameFormat("guava-%d");

    // 明确线程池的关键状态
    // 阿里巴巴 Java开发手册：不推荐使用Executors，不利于开发明确线程池的运行规则，规避资源耗尽的风险。
    ThreadPoolExecutor threadPoolExecutor =
        new ThreadPoolExecutor(10, 12, 3, TimeUnit.SECONDS, queue, threadFactoryBuilder.build());

    // guava的线程池封装类
    ListeningExecutorService listeningExecutorService =
        MoreExecutors.listeningDecorator(threadPoolExecutor);

    ListenableFuture<Integer> listenableFuture = listeningExecutorService.submit(() -> {
      System.out.println("callable ID = " + Thread.currentThread().getId());
      System.out.println("callable Name = " + Thread.currentThread().getName());
      return 0;
    });

    // 只是拒绝新增任务，在队列中的依旧会执行
    listeningExecutorService.shutdown();

    // 注册监听器。异步调用完成时会在指定的线程池（可以为不同线程池）中执行注册的监听器
    // 当使用 FutureTask 作为入参时，listenableFuture.get 返回的是 null，虽然回调方法会依旧执行到
    listenableFuture.addListener(() -> {
      System.out.println("addListener ID = " + Thread.currentThread().getId());
      System.out.println("addListener Name = " + Thread.currentThread().getName());
    }, Executors.newSingleThreadExecutor());


    // 当需要操作返回值得时候。异步调用完成时会在指定的线程池（可以为不同线程池）中执行注册的监听器
    Futures.addCallback(listenableFuture, new FutureCallback<Integer>() {
      @Override
      public void onFailure(Throwable ex) {
        System.out.println("addCallback is onFailure. Throwable = " + ex);
        System.out.println("addCallback ID = " + Thread.currentThread().getId());
        System.out.println("addCallback Name = " + Thread.currentThread().getName());
      }

      @Override
      public void onSuccess(Integer result) {
        System.out.println("addCallback is onSuccess. result = " + result);
        System.out.println("addCallback ID = " + Thread.currentThread().getId());
        System.out.println("addCallback Name = " + Thread.currentThread().getName());
      }
    }, Executors.newSingleThreadExecutor());

    // 调用顺序：listenableFuture.callable -> listenableFuture.addListener -> listenableFuture.get
    // -> Futures.addCallback
    try {
      // 此处采用阻塞的方式获取结果(可设置超时)，可以也可以通过 listenableFuture.isDone() 判断是否先执行其它业务
      Integer result = listenableFuture.get();
      System.out.println("listenableFuture.get, result = " + result);
    } catch (InterruptedException e) {
      System.out.println("get result is InterruptedException. InterruptedException = " + e);
    } catch (ExecutionException e) {
      System.out.println("get result is InterruptedException. ExecutionException = " + e);
    }

    // 执行效果：
    // callable ID = 11
    // callable Name = guava-0
    // addListener ID = 12
    // addListener Name = pool-2-thread-1
    // listenableFuture.get, result = 0
    // addCallback is onSuccess. result = 0
    // addCallback ID = 13
    // addCallback Name = pool-3-thread-1
  }
}
```

### 功能调测

```
2018/09/10
尝试:单线程池, 3个已经commit的线程, sleep 3s。
预期: futureTask.get时设置4s不超时。如果是commit开始是就计时的,必然获取超时,只有从get后开始计时才不会超时。
结果:三个线程都未超时
```

## 参考链接

### FutureTask

https://blog.csdn.net/linchunquan/article/details/22382487

https://blog.csdn.net/hbtj_1216/article/details/70569881

https://blog.csdn.net/xiaomingdetianxia/article/details/77774637

https://blog.csdn.net/zmx729618/article/details/51596414

源码分析

https://www.jianshu.com/p/06f8df545a86?winzoom=1

### ListenableFuture

https://github.com/google/guava

http://ifeve.com/google-guava/

Guava并发：ListenableFuture使用介绍以及示例

http://outofmemory.cn/java/guava/concurrent/ListenableFuture

ThreadFactoryBuilder

https://www.jianshu.com/p/b94a57bd5eb9

ListenableFuture

https://blog.csdn.net/jackyechina/article/details/52710783

Guava介绍

https://blog.csdn.net/axi295309066/article/details/53726023