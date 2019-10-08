---
title: 线程间通信
date: 2019-10-07 22:45:38
tags: Java
---

## ~~suspendd & resume~~
开始之前，先来看一下suspend和resume的使用方法。

假设有以下场景：小朋友到商店买冰激凌，此时恰好店里没有，老板需要先制作，小朋友等待。等老板制作好后通知小朋友。

用代码表示如下：
```
public class SuspendResumeTest {
    static Object ice = null;
    public static void main(String[] args) throws InterruptedException {
        new SuspendResumeTest().test1();
    }

    public void test1() throws InterruptedException {
        Thread consumer = new Thread(() -> {
            System.out.println("小朋友来买冰激凌");
            if (ice == null) {
                System.out.println("没有冰激凌，进入等待");
                Thread.currentThread().suspend();
                System.out.println("等待完成，小朋友买到了冰激淋");
            }
        });
        consumer.start();
        Thread.sleep(2000);
        ice = new Object();
        System.out.println("有了冰激凌，通知小朋友");
        consumer.resume();
    }
}
```
输出结果如下：
```
小朋友来买冰激凌
没有冰激凌，进入等待
有了冰激凌，通知小朋友
等待完成，小朋友买到了冰激淋
```

看着没有问题，但是如果小朋友等的不耐烦了，回家关上门睡觉了（加锁），我们在看一下:
```
    public void test2() throws InterruptedException {
        Thread consumer = new Thread(() -> {
            System.out.println("小朋友来买冰激凌");
            if (ice == null) {
                System.out.println("没有冰激凌，进入等待");
                synchronized (SuspendResumeTest.class) {
                    // 小朋友关上门睡觉（加锁）
                    Thread.currentThread().suspend();
                }
                System.out.println("等待完成，小朋友买到了冰激淋");
            }
        });
        consumer.start();

        Thread.sleep(2000);
        ice = new Object();
        System.out.println("有了冰激凌，通知小朋友");
        synchronized (SuspendResumeTest.class) {
            // 由于小朋友没有释放锁，这里拿不到锁，永远通知不到小朋友
            consumer.resume();
        }
    }
```
可以看到，consumer线程在挂起时拿到了当前对象的锁，主线程在通知consumer线程时由于拿不到锁，consumer线程一直处于挂起状态，而主线程一直阻塞。

导致这种问题的根本原因是：
**线程在执行suspend时，是不释放锁的。**

再来看一种场景：主线程先执行resume，子线程后执行suspend
```
    public void test3() throws InterruptedException {
        Thread consumer = new Thread(() -> {
            System.out.println("小朋友来买冰激凌");
            if (ice == null) {
                System.out.println("没有冰激凌，进入等待");
                try {
                    Thread.sleep(6000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                Thread.currentThread().suspend();
                System.out.println("等待完成，小朋友买到了冰激淋");
            }
        });
        consumer.start();

        Thread.sleep(2000);
        ice = new Object();
        System.out.println("有了冰激凌，通知小朋友");
        consumer.resume();
    }
```
执行结果：
```
小朋友来买冰激凌
没有冰激凌，进入等待
有了冰激凌，通知小朋友
```
可以看到：由于主线程先调用了子线程的resume方法，而子线程后来才suspend，导致子线程没有被唤醒。
结论：**线程的resume必须在suspend之后才能正确唤醒线程**

通过上面的例子可以知道：
1. 线程在执行suspend时，是不释放锁的
2. 线程的resume必须在suspend之后才能正确唤醒线程，对2个方法的调用顺序有关
这2个问题都有可能导致死锁，因此suspend和resume已被弃用。

## wait & notify

wait方法导致当前线程等待，加入该对象的等待集合中，并且放弃当前持有的对象锁。
notify/notifyAll唤醒一个或所有正在等待这个对象所的线程。


示例1:
先看看wait方法不再同步代码块里：
```
    public void test1() {
        try {
            wait();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
```
执行结果：
```
Exception in thread "main" java.lang.IllegalMonitorStateException
	at java.lang.Object.wait(Native Method)
	at java.lang.Object.wait(Object.java:502)
	at com.jadson.study.common.thread.WaitNotifyTest.test1(WaitNotifyTest.java:10)
	at com.jadson.study.common.thread.WaitNotifyTest.main(WaitNotifyTest.java:5)
```
由于wait不在同步代码块中，抛出了异常。


正确的用法:
```
    public void test2() {
        Thread consumer = new Thread(() -> {
            System.out.println("小朋友来买冰激凌");
            if (ice == null) {
                synchronized (this) {
                    System.out.println("没有冰激凌，进入等待");
                    try {
                        this.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println("等待完成，小朋友买到了冰激淋");
                }
            }
        });
        consumer.start();

        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        ice = new Object();
        System.out.println("有了冰激凌，通知小朋友");
        synchronized (this) {
            // notify和wait都必须在synchronized同步块中
            // 且2者必须是同一个对象所
            this.notifyAll();
        }
    }
```
执行结果：
```
小朋友来买冰激凌
没有冰激凌，进入等待
有了冰激凌，通知小朋友
等待完成，小朋友买到了冰激淋
```

结论：
1. 虽然wait会自动解锁，但是对顺序有要求。如果在notify被调用之后，才开始wait方法的调用，线程会永远处于WAITING状态。类似于suspend和resume。
2. 这些方法只能由同一对象锁的持有者线程调用，也就是写在同步代码块里，否则会抛出IllegalMonitorStateException异常。
3. wait和notify都必须在synchronized同步块中
4. 调用wait方法后会释放锁

> wait与sleep的区别：
> 1. wait会释放锁，sleep不释放锁
> 2. wait调用后需要调用notify唤醒，sleep在时间到后继续执行，不需要唤醒
> 3. wait被唤醒后需要再次等待CPU资源，sleep时间到后直接继续执行
> 4. wait需要在synchronized同步块中，sleep不需要

## park & unpark

{% asset_img java-thread-communicate-1.png %}
线程调用park则等待“许可”，unpark方法为指定线程提供“许可证”。

用法：
```
    public void test() {
        Thread consumer = new Thread(() -> {
            System.out.println("小朋友来买冰激凌");
            if (ice == null) {
                System.out.println("没有冰激凌，进入等待");
                LockSupport.park();
                System.out.println("等待完成，小朋友买到了冰激淋");
            }
        });
        consumer.start();

        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        ice = new Object();
        System.out.println("有了冰激凌，通知小朋友");
        // 需要指定要唤醒的线程
        LockSupport.unpark(consumer);
    }
```
输出：
```
小朋友来买冰激凌
没有冰激凌，进入等待
有了冰激凌，通知小朋友
等待完成，小朋友买到了冰激淋
```

那么，线程调用park还没调用unpark这段时间内，有没有释放锁呢，来看一下：
```
    public void test() {
        Thread consumer = new Thread(() -> {
            System.out.println("小朋友来买冰激凌");
            if (ice == null) {
                System.out.println("没有冰激凌，进入等待");
                synchronized (this) {
                    LockSupport.park();
                }
                System.out.println("等待完成，小朋友买到了冰激淋");
            }
        });
        consumer.start();

        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        ice = new Object();
        System.out.println("有了冰激凌，通知小朋友");
        synchronized (this) {
            LockSupport.unpark(consumer);
        }
    }
```
输出：
```
小朋友来买冰激凌
没有冰激凌，进入等待
有了冰激凌，通知小朋友
```
可以看到，主线程在执行unpark时一直获取不到锁，可见**consumer线程执行park后并没有释放锁**。

结论：
1. 调用unpark()方法之后，再调用park()，线程会直接运行。
2. 提前调用的unpark，连续多次调用unpark后，第一次调用park后会拿到许可直接运行，后续调用会进入等待。
3. 线程调用park还未调用unpark挂起后，是不释放锁的。

## 拓展--伪唤醒

之前代码中用if来判断是否需要进入等待（挂起）状态，这样的做法是错误的！

官方建议应该在循环中检查等待条件，原因是处于等待状态的线程可能会收到错误警报和伪唤醒，如果不在循环中检查等待条件，程序就会在没有满足结束条件的情况下退出。

伪唤醒是指线程并非是因为notify、notifyAll、resume、unpark等api调用而意外唤醒，是更底层原因导致。

错误的用法：
```
if (ice == null) {
    System.out.println("没有冰激凌，进入等待");
    LockSupport.park();
    System.out.println("等待完成，小朋友买到了冰激淋");
}
```

正确的用法：
```
while (ice == null) {
    System.out.println("没有冰激凌，进入等待");
    LockSupport.park();
    System.out.println("等待完成，小朋友买到了冰激淋");
}
```