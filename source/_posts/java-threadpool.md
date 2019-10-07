---
title: 线程池ThreadPool
date: 2019-10-07 14:12:52
tags: Java
---

```
public ThreadPoolExecutor(int corePoolSize,
                            int maximumPoolSize,
                            long keepAliveTime,
                            TimeUnit unit,
                            BlockingQueue<Runnable> workQueue,
                            RejectedExecutionHandler handler) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
            Executors.defaultThreadFactory(), handler);
}
```
+ corePoolSize：核心线程数量
+ maximumPoolSize：最大线程数量
    > 1.包含了核心线程数，例如maximumPoolSize=10，corePoolSize=5，则只能创建10-5=5个非核心线程
    > 2.如果工作队列无界，则永远不会用到非核心线程
+ keepAliveTime：线程的超时时间，若allowCoreThreadTimeOut=true，则核心线程达到超时时间后也会被清除
+ unit：超时时间的单位
+ workQueue：工作队列
+ handler：线程池满后的处理策略，默认直接抛出RejectedExecutionException异常

{% asset_img java-threadpool-1.png %}

可以自己实例化线程池，也可以用Executors创建线程池的工厂类，常用方法如下：
+ ExecutorService newFixedThreadPool(int nThreads)： 创建一个固定大小、任务队列容量无界的线程池。核心线程数=最大线程数。
+ ExecutorService newCachedThreadPool()：创建一个大小无界的缓冲线程池。它的任务队列是一个同步队列。任务加入到池中，如果池中有空闲线程，则用空闲线程执行，如无则创建新线程执行。池中的线程空闲超过60秒，将被销毁释放。线程数随任务的多少变化。适用于执行耗时较小的异步任务。池的核心线程数=0，最大线程数=Integer.MAX_VALUE.
+ ExecutorService newSingleThreadExecutor()：只有一个线程来执行无界任务队列的单一线程池。该线程池确保任务按加入的顺序一个一个一次执行。当唯一的线程因任务异常终止时，将创建一个新的线程来继续执行后续的任务。与newFixedThreadPool区别在于，单一线程池的大小再newSingleThreadExecutor方法中硬编码，不能改变。
+ ScheduledExecutorService newScheduledThreadPool(int corePoolSize)：能定时执行任务的线程池。该池的核心线程数由参数指定，最大线程数=Integer.MAX_VALUE

**如何确定合适的线程数量？**
+ 计算型任务： CPU的1～2倍
+ IO型任务：相对比计算型任务，需多一些线程，要根据具体的IO阻塞时长进行考量决定。也可考虑根据需要在一个最小数量和最大数量间自动增减线程数。
> 如tomgcat中默认的最大线程数为：200