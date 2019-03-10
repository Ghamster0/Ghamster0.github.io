---
title: Java中wait、notify与notifyAll
tags:
- Java
- 多线程
date: 2019-03-10 15:43:03
---

## 概述  
Java中可使用`wait`和`notify`(或`notifyAll`)方法同步对临界资源的访问  
这些方法在Object类中定义，因此可以在任何对象上调用  
在对象上调用`wait`方法使线程阻塞，在对象上调用`notify`或`notifyAll`会唤醒之前在该对象上调用`wait`阻塞的线程  
调用这些方法之前需要线程已经获取对象的锁（This method should only be called by a thread that is the owner of this object's monitor），否则会抛出`java.lang.IllegalMonitorStateException`。因此只能在同步方法或同步代码块中使用

<!-- more -->

## wait  
- 阻塞当前线程，释放锁
- `wait()`或`wait(0)`，为无限期阻塞（两者完全相同）；`wait(long timeoutMillis)`或`wait(long timeoutMillis, int nanos)`,若在参数指定时间内没有被唤醒或打断，自动恢复执行
- 可以被`notify`或`notifyAll`唤醒
- 可以被`interrupt`方法打断，抛出`InterruptedException`  

## notify & notifyAll  
- `notify`: 唤醒一个在该对象上调用`wait`方法阻塞的线程
- `notifyAll`: 唤醒所有在该对象上调用`wait`方法阻塞的线程  

## notify与notifyAll测试  
`notify`相对于`notifyAll`方法是一种性能优化，因为`notify`只会唤醒一个线程，但`notifyAll`会唤醒所有等待的线程，使他们竞争cpu；但同时，使用`notify`你必须确定被唤醒的是合适的线程  

> 下面的测试代码展示了“必须唤醒合适线程的问题”  

- `Critical`类只包含一个`Color`类的对象，通过对象初始化语句赋值为`Color.B`  
- `ColorModifier`类实现了`Runnable`接口，包含三个域： `critical`、`target`和`to`，操作`critical`的`color`对象，当与目标颜色`target`相符时，将颜色修改为`to`指定的值。
- `main`方法中，依次创建三个`ColorModifier`类的实例，分别为**R->G**，**G->B**，**B->R**，交给`ExectorService`执行，30s后关闭`ExectorService`，三个线程收到`InterruptedException`退出  

使用`notifyAll`的测试代码如下：  

```Java
package main.test;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class TestNotify {

    enum Color {R, G, B}

    private static class Critical {
        public Color color = Color.R;
    }

    private static class ColorModifier implements Runnable {
        private Critical critical;
        private Color target;
        private Color to;

        public ColorModifier(Critical critical, Color target, Color to) {
            this.critical = critical;
            this.target = target;
            this.to = to;
        }

        @Override
        public void run() {
            System.out.printf("-> Thread start: Modifier %s to %s\n", target, to);
            try {
                while (!Thread.interrupted()) {
                    synchronized (critical) {
                        while (critical.color != target) {
                            System.out.printf("  - Wait: Modifier %s -> %s, Current color: %s\n", target, to, critical.color);
                            critical.wait();
                            System.out.printf("  + Resume from wait: Modifier %s -> %s, Current color: %s\n", target, to, critical.color);
                        }
                        //change critical.color and notify others
                        critical.color = to;
                        System.out.printf("\n>>> Color changed: %s to %s!\n", target, to);
                        TimeUnit.SECONDS.sleep(1);
                        critical.notifyAll();
                    }
                }
            } catch (InterruptedException e) {
                System.out.printf("Thread Modifier %s -> %s exit!\n", target, to);
            }

        }

        public static void main(String[] args) throws InterruptedException {
            ExecutorService exec = Executors.newCachedThreadPool();
            Critical c = new Critical();
            exec.execute(new ColorModifier(c, Color.R, Color.G));
            exec.execute(new ColorModifier(c, Color.G, Color.B));
            exec.execute(new ColorModifier(c, Color.B, Color.R));
            TimeUnit.SECONDS.sleep(30);
            exec.shutdownNow();
        }
    }
}
```

输出如下：  

```console
-> Thread start: Modifier R to G

>>> Color changed: R to G!
-> Thread start: Modifier B to R
-> Thread start: Modifier G to B
  - Wait: Modifier R -> G, Current color: G

>>> Color changed: G to B!
  - Wait: Modifier G -> B, Current color: B

>>> Color changed: B to R!
  - Wait: Modifier B -> R, Current color: R
  + Resume from wait: Modifier G -> B, Current color: R
  - Wait: Modifier G -> B, Current color: R
  + Resume from wait: Modifier R -> G, Current color: R

>>> Color changed: R to G!
  - Wait: Modifier R -> G, Current color: G
  + Resume from wait: Modifier B -> R, Current color: G
  - Wait: Modifier B -> R, Current color: G
  + Resume from wait: Modifier G -> B, Current color: G

>>> Color changed: G to B!
  - Wait: Modifier G -> B, Current color: B
  + Resume from wait: Modifier R -> G, Current color: B
  - Wait: Modifier R -> G, Current color: B
  + Resume from wait: Modifier B -> R, Current color: B

>>> Color changed: B to R!
... ...
Thread Modifier B -> R exit!
Thread Modifier R -> G exit!
Thread Modifier G -> B exit!

Process finished with exit code 0
```

任意时刻，系统中有三个`ColorModifier`的线程（更严谨的表述是：target为ColorModifer对象的线程）RtoG、GtoB和BtoR，假设RtoG修改颜色后（console第17行），调用`notifyAll`方法，使GtoB、BtoR线程被唤醒，三个线程均可开始（继续）执行。当前颜色为`Color.G`，执行至代码32行，RtoG和BtoR调用`wait`阻塞，GtoB修改颜色并调用`notifyAll`方法，如此往复  

测试`notify`方法时，将第40行代码修改为`critical.notify();`，输出如下：  

```console
-> Thread start: Modifier B to R
-> Thread start: Modifier R to G
-> Thread start: Modifier G to B
  - Wait: Modifier B -> R, Current color: R
  - Wait: Modifier G -> B, Current color: R

>>> Color changed: R to G!
  - Wait: Modifier R -> G, Current color: G
  + Resume from wait: Modifier B -> R, Current color: G
  - Wait: Modifier B -> R, Current color: G
Thread Modifier B -> R exit!
Thread Modifier R -> G exit!
Thread Modifier G -> B exit!

Process finished with exit code 0
```

每次运行测试得到的输出各不相同，但几乎所有的测试都会导致死锁，直到时间耗尽，调用`ExectorService.shutdownNow()`结束程序。以本次运行结果为例，RtoG、GtoB和BtoR依次启动，`Critical`对象初始颜色为`Color.R`。执行至代码32行，BtoR和GtoB调用`wait`阻塞（对应console第4-5行）；RtoG将颜色修改为`Color.G`，调用`notify`方法，BtoR被唤醒；RtoG继续执行，经过代码32行判断后调用`wait`阻塞；BtoR被唤醒后，经过32行同样调用`wait`阻塞 -- 至此三个线程全部阻塞，程序陷入死锁。  

> 对于本程序而言，“合适的线程”是指：BtoR的notify必须唤醒RtoG，RtoG的notify必须唤醒GtoB，GtoB的notify必须唤醒BtoR

## one more thing  
如果对测试代码稍作修改会发生有趣的事情：
1. 将`Critical`对象的`color`属性初始值设为`Color.B`（12行）
2. 在`main`方法的每个`exec.execute()`方法后插入`TimeUnit.SECONDS.sleep(1);`，插入后代码如下：  
```java
public static void main(String[] args) throws InterruptedException {
    ExecutorService exec = Executors.newCachedThreadPool();
    Critical c = new Critical();
    exec.execute(new ColorModifier(c, Color.R, Color.G));
    TimeUnit.SECONDS.sleep(1);
    exec.execute(new ColorModifier(c, Color.G, Color.B));
    TimeUnit.SECONDS.sleep(1);
    exec.execute(new ColorModifier(c, Color.B, Color.R));
    TimeUnit.SECONDS.sleep(30);
    exec.shutdownNow();
}
```
此时会得到如下输出:
```console
>>> Color changed: B to R!
  - Wait: Modifier B -> R, Current color: R
  + Resume from wait: Modifier R -> G, Current color: R

>>> Color changed: R to G!
  - Wait: Modifier R -> G, Current color: G
  + Resume from wait: Modifier G -> B, Current color: G

>>> Color changed: G to B!
  - Wait: Modifier G -> B, Current color: B
  + Resume from wait: Modifier B -> R, Current color: B
```
程序并未出现死锁！似乎BtoR的`notify`总会唤醒RtoG，RtoG会唤醒GtoB，GtoB会唤醒BtoR
换言之，`notify`被调用时，唤醒的线程不是随机的，而是所有阻塞的线程中，最早调用`wait`的那个

### 测试  
> 测试环境：window x64，jdk11  

- 内部类`WaitAndNotify`实现了`Runnable`接口，构造方法需要传入一个`Object`对象（o）
- 在`run`方法中，首先调用`o.wait()`阻塞，被唤醒后调用`o.notify()`
- `main`方法依次产生`THREAD_NUMBERS`个使用`WaitAndNotify`对象创建的线程，并交由`ExecutorService`执行；在`main`方法中调用`notify`，引发链式反应，使所有线程依次执行
- 使用`CountDownLatch`计数，所有线程完成后，关闭`ExecutorService`退出程序  

测试代码如下：
```java
package main.test;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class TestSynchronizedLockOrder {

    private static final int THREAD_NUMBERS = 5;

    private static class WaitAndNotify implements Runnable {
        private static int count = 0;
        private int id = count++;
        private CountDownLatch countDownLatch;
        private Object o;

        public WaitAndNotify(Object o, CountDownLatch c) {
            this.o = o;
            this.countDownLatch = c;
        }

        @Override
        public void run() {
            synchronized (o) {
                try {
                    System.out.println("WAN id=" + id + " call wait");
                    o.wait();
                    TimeUnit.SECONDS.sleep(1);
                    System.out.println("WAN id=" + id + " running");
                    o.notify();
                    System.out.println("WAN id=" + id + " call notify");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            countDownLatch.countDown();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Object o = new Object();
        CountDownLatch countDownLatch = new CountDownLatch(THREAD_NUMBERS);
        ExecutorService e = Executors.newCachedThreadPool();
        for (int i = 0; i < THREAD_NUMBERS; i++) {
            e.execute(new WaitAndNotify(o, countDownLatch));
            TimeUnit.SECONDS.sleep(1);
        }
        System.out.println("===================\nAll thread started!\n===================");
        synchronized (o) {
            o.notify();
        }
        countDownLatch.await();
        e.shutdownNow();
    }
}
```
程序输出如下：
```console
WAN id=0 call wait
WAN id=1 call wait
WAN id=2 call wait
WAN id=3 call wait
WAN id=4 call wait
===================
All thread started!
===================
WAN id=0 running
WAN id=0 call notify
WAN id=1 running
WAN id=1 call notify
WAN id=2 running
WAN id=2 call notify
WAN id=3 running
WAN id=3 call notify
WAN id=4 running
WAN id=4 call notify

Process finished with exit code 0
```
### 结论  
显然，在本平台上调用`notify`方法时，被唤醒的永远是最早调用`wait`方法阻塞的线程，但这个结论是否具有普遍性？

jdk文档对于notify的描述如下：
> Wakes up a single thread that is waiting on this object's monitor. If any threads are waiting on this object, one of them is chosen to be awakened. The choice is arbitrary and occurs at the discretion of the implementation...
> The awakened thread will not be able to proceed until the current thread relinquishes the lock on this object. The awakened thread will compete in the usual manner with any other threads that might be actively competing to synchronize on this object...  

参考jdk文档的内容，总结来说有两点：
1. 调用`notify`方法会唤醒一个阻塞的线程，且这个线程是随机的，且不同平台可以有不同实现
2. 被唤醒的线程需要竞争临界资源，相比于其他线程不具有更高或更低的优先级

因此，这种测试结果只能算平台的特例……

《全剧终》