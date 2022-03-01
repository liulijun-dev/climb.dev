---
title: Semaphore和线程池的差异
date: 2020-03-05 13:27:46
tags: Java,Semaphore,TheadPool
---
# 1. 什么是Semaphore和线程池

**`Semaphore`**称为信号量，是`java.util.concurrent`一个并发工具类，用来控制可同时并发的线程数，其内部维护了一组虚拟许可，通过构造器指定许可的数量。线程在执行时，需要通过`acquire()`获得许可后才能执行，如果无法获得许可，则线程将一直等待；线程执行完后需要通过`release()`释放许可，以使得其他线程可以获得许可。

**线程池**也是一种控制任务并和执行的方式，通过线程复用的方式来减小频繁创建和销毁线程带来的开销。一般线程池可同时工作的线程数量是一定的，超过该数量的线程需进入线程队列等待，直到有可用的工作线程来执行任务。

使用Seamphore，一般是创建了多少线程，实际就会有多少线程并发执行，只是可同时执行的线程数量会受到信号量的限制。但使用线程池，创建的线程只是作为任务提交给线程池执行，实际工作的线程由线程池创建，并且实际工作的线程数量由线程池自己管理。

# 2. Semaphore和线程池的区别

先亮结果，Semaphore和线程池的区别如下：

- 使用`Semaphore`，实际工作线程由开发者自己创建；使用线程池，实际工作线程由线程池创建
- 使用`Semaphore`，并发线程的控制必须手动通过`acquire()`和`release()`函数手动完成；使用线程池，并发线程的控制由线程池自动管理
- 使用`Semaphore`不支持设置超时和实现异步访问；使用线程池则可以实现超时和异步访问，通过提交一个`Callable`对象获得`Future`，从而可以在需要时调用`Future`的方法获得线程执行的结果，同样利用`Future`也可以实现超时

接下来用示例说明结果：

**1. 使用Semaphore**

```java
public static void testSemaphore() {
    Semaphore semaphore = new Semaphore(2);
    for (int i = 0; i < 6; i++) {
        Thread thread = new Thread() {
            public void run() {
                try {
                    semaphore.acquire();
                    System.out.println(Thread.currentThread().getName() + " start running");
                    Thread.sleep(2000);
                    System.out.println(Thread.currentThread().getName() + " stop running");
                    semaphore.release();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };
        thread.setName("Semaphore thread " + i);
        thread.start();
    }
}
```

输出结果如下：

```latex
Semaphore thread 0 start running
Semaphore thread 1 start running
Semaphore thread 0 stop running
Semaphore thread 1 stop running
Semaphore thread 2 start running
Semaphore thread 3 start running
Semaphore thread 3 stop running
Semaphore thread 2 stop running
Semaphore thread 4 start running
Semaphore thread 5 start running
Semaphore thread 5 stop running
Semaphore thread 4 stop running
```

通过输出可以发现：

- 每次最多打印两个start running记录，因为`Seamphore`的信号量是2
- 只有当其他线程释放了`Seamphore`后，新的线程才能开始执行
- 线程的名字以 Semaphore thread开头，且每个执行线程的名字都不相同

**2. 使用线程池** 

```java
public static void testThreadPool() {
    ExecutorService executorService = new ThreadPoolExecutor(2, 5,
            0L, TimeUnit.MILLISECONDS,
            new LinkedBlockingQueue<Runnable>());

    for (int i = 0; i < 6; i++) {
        Thread thread = new Thread() {
            public void run() {
                try {
                    System.out.println(Thread.currentThread().getName() + " start running");
                    Thread.sleep(2000);
                    System.out.println(Thread.currentThread().getName() + " stop running");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };
        thread.setName("ThreadPool thread " + i);
        executorService.submit(thread);
    }
    executorService.shutdown();
}
```

输出结果如下：

```latex
pool-1-thread-2 start running
pool-1-thread-1 start running
pool-1-thread-1 stop running
pool-1-thread-2 stop running
pool-1-thread-2 start running
pool-1-thread-1 start running
pool-1-thread-2 stop running
pool-1-thread-1 stop running
pool-1-thread-2 start running
pool-1-thread-1 start running
pool-1-thread-1 stop running
pool-1-thread-2 stop running
```

通过输出可以发现：

- 每次最多打印两个start running记录，因为线程池的核心容量是2，多余的线程任务放到阻塞队列等待
- 两个线程的名字，即pool-1-thread-1和pool-1-thread-2，不是开发者自己创建的，而是线程池创建的
- 任务的执行和结束都是由线程池来完成，开发者只需要将任务提交给线程池即可

# 3. 用Semaphore实现互斥锁

使用信号值为1的`Semaphore`对象便可以实现互斥锁，示例如下：

```java
public static void testSemaphoreMutex() {
    Semaphore semaphore = new Semaphore(1);
    for (int i = 0; i < 6; i++) {
        new Thread() {
            public void run() {
                try {
                    semaphore.acquire();
                    System.out.println(Thread.currentThread().getName() + " get semaphore");
                    Thread.sleep(2000);
                    System.out.println(Thread.currentThread().getName() + " release semaphore");
                    semaphore.release();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }.start();
    }
}
```

输出结果如下：

```latex
Thread-0 get semaphore
Thread-0 release semaphore
Thread-1 get semaphore
Thread-1 release semaphore
Thread-2 get semaphore
Thread-2 release semaphore
Thread-3 get semaphore
Thread-3 release semaphore
Thread-4 get semaphore
Thread-4 release semaphore
Thread-5 get semaphore
Thread-5 release semaphore
```

可以看出，任何一个线程在释放许可之前，其它线程都拿不到许可。这样当前线程必须执行完毕，其它线程才可执行，这样就实现了互斥。

# 4. Semaphore的易错点

使用`Semophore`时有一个非常容易犯错误的地方，即**先release再acqure**后会导致`Semophore`管理的虚拟许可额外新增一个，示例如下：

```
public static void firstReleaseThenAcquire() throws InterruptedException {
    Semaphore semaphore = new Semaphore(1);
    System.out.println("Init permits: " + semaphore.availablePermits());
    semaphore.release();
    System.out.println("Permits after first releasing:" + semaphore.availablePermits());
    semaphore.acquire();
    System.out.println("Permits after first acquiring:" + semaphore.availablePermits());
    semaphore.acquire();
    System.out.println("Permists after second acquiring:" + semaphore.availablePermits());
}
```

输出结果如下：

```latex
Init permits: 1
Permits after first releasing:2
Permits after first acquiring:1
Permists after second acquiring:0
```

可以发现，虽然`Semophore`的初始信号量为1，但是当先调用`release()`后，`Semophore`的信号量变为2了，因此才能够连续调用两次`acquire()`都能获得许可。
