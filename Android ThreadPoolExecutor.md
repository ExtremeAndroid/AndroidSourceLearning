# Android Thread
## 1. ThreadPoolExecutor

### 1.1 概述

ThreadPoolExecutor接收到新任务时候，执行规则如下：

- 若池中活动的线程数量 < corePoolSize，则直接启动核心线程。
- 若池中活动的线程数量 >= corePoolSize，则放入队列workQueue中。
- 若队列workQueue已满，且池中活动的线程数量 < maximumPoolSize，则启动非核心线程执行任务。
- 若达到了maximumPoolSize，则任务被拒绝。调用RejectedExecutionHandler#rejectedExecution(Runnable r, ThreadPoolExecutor executor)通知调用者任务被拒绝。
- 调用者可以更改RejectedExecutionHandler策略来防止不崩溃。

### 1.2 构造方法

1. 初始化配置线程池。
2. corePoolSize：线程池核心线程数。默认情况下，不管是否闲置，都不会关闭。若allowCoreThreadTimeOut设置为true，则等待新任务的过程中有超时策略。等待keepAliveTime，然后超时。
3. maximumPoolSize：线程池所能容纳的最大活动线程数。当活动线程到达此数后，之后的新任务会被阻塞。
4. keepAliveTime：非核心线程闲置的超时时长。活动线程数量>核心线程数，则若有线程执行完毕任务后，闲置等待新任务的过程中，超时判断时间。allowCoreThreadTimeOut为true时候，核心线程也会超时。
5. unit：超时时长单位
6. workQueue：线程池中任务队列，通过execute提交的Runnable对象保存在此队列中
7. threadFactory：创建新线程的工厂
8. handler：任务因为达到了线程边界和队列容量而被阻止时的处理程序，会通过RejectedExecutionHandler 通知出来。默认使用 AbortPolicy，可以自定义拦截。


```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
  // code ...
}
```

### 1.3 几种常见封装好的线程池

Android提供了几种不同特种常见的线程池，都是通过配置ThreadPoolExecutor不同参数实现的。
#### 1.3.1 FixedThreadPool

1. 显而易见，核心线程数==最大线程数。则可以活动的线程数量一定。
2. 当线程均活动处理任务，则新来的线程只能排队等待。
3. 当线程均空闲，也不会被回收。
4. 长期保持空闲线程不被回收，则新来任务可以快速响应。
5. keepAliveTime==0，但是没有非核心线程。核心线程没有超时机制。
6. 任务队列没有大系限制。

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```
#### 1.3.2 CachedThreadPool

1. 核心线程数\==0，最大线程数量\==Integer.MAX_VALUE，近似无限大。则每来一个新任务，若没有等待的线程，则会创建一个非核心线程来执行任务。
2. 队列不会被排队。
3. 任务执行完毕后，线程等待新任务的时间为60s，超时后则回收。
4. 综上，心来任务立即执行，执行完毕快速回收。可以应对突发性的、执行耗时短，任务量大的情况，性能很高。

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```
#### 1.3.3 ScheduledThreadPool

1. 核心线程数固定，最大线程数量==Integer.MAX_VALUE，近似无限大，等待时间为0，则一执行完毕就回收。
2. 适合执行延时任务or周期性任务

```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE,
          0L, MILLISECONDS,
          new DelayedWorkQueue());
}
```
#### 1.3.4 SingleThreadExecutor

1. 核心线程数\==最大线程数\==1。只有一个线程活跃。
2. 新添加进来任务，若已有任务在执行，就放在队列中等待。
3. 每次执行一个线程，保证了所有的任务按顺序执行。在外界可以不需要考虑线程同步的问题。
4. 若当前的线程出现问题退出了，则会创建一个新的线程继续没有完成的任务。

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```
### 1.4 排队策略
线程池的排队策略与BlockingQueue有关。常见的排队策略如下：
#### 1.4.1 ArrayBlockingQueue
- 基于数组的缓冲队列，需要指定大小。
2. 接收到任务的时候，如果没有达到corePoolSize的值，则新建线程(核心线程)执行任务。
3. 如果达到了，则入队等候。
4. 如果队列已满，则新建线程(非核心线程)执行任务。
5. 又如果总线程数到了maximumPoolSize，并且队列也满了，则发生错误

#### 1.4.2 LinkedBlockingQueue
- 基于链表的缓冲队列。
2. 这个队列接收到任务的时候，如果当前线程数小于核心线程数，则新建线程(核心线程)处理任务；
3. 如果当前线程数等于核心线程数，则进入队列等待。
4. 由于这个队列没有最大值限制，即所有超过核心线程数的任务都将被添加到队列中，这也就导致了maximumPoolSize的设定失效，因为总线程数永远不会超过corePoolSize。

#### 1.4.3 SynchronousQueue
- 这个队列接收到任务的时候，会直接提交给线程处理，而不保留它。
2. 如果所有线程都在工作怎么办？那就新建一个线程来处理这个任务！
3. 所以为了保证不出现<线程数达到了maximumPoolSize而不能新建线程>的错误，使用这个类型队列的时候，maximumPoolSize一般指定成Integer.MAX_VALUE，即无限大。

#### 1.4.4 DelayQueue
- 队列内元素必须实现Delayed接口，这就意味着你传进去的任务必须先实现Delayed接口。
2. 这个队列接收到任务时，首先先入队，只有达到了指定的延时时间，才会执行任务

### 1.5 拒绝策略
拒绝策略都继承自RejectedExecutionHandler，预定义的拒绝策略如下：

#### 1.5.1 CallerRunsPolicy
1. 由调用线程处理该任务
2. 若调用线程已经关闭，则丢弃此任务。

```java
public static class CallerRunsPolicy implements RejectedExecutionHandler {
    public CallerRunsPolicy() { }

    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            r.run();
        }
    }
}
```
#### 1.5.2 AbortPolicy
1. 丢弃任务
2. 抛出RejectedExecutionException异常

```java
public static class AbortPolicy implements RejectedExecutionHandler {
    public AbortPolicy() { }

    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        throw new RejectedExecutionException("Task " + r.toString() + " rejected from " + e.toString());
    }
}

```
#### 1.5.3 DiscardPolicy
1. 默默地丢弃任务
2. 不抛出异常，对调用线程没有影响。 

```java
public static class DiscardPolicy implements RejectedExecutionHandler {
    public DiscardPolicy() { }

    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    }
}
```
#### 1.5.4 DiscardOldestPolicy
1. 丢弃队列最前面，没有被执行的任务
2. 然后重新从队列中拉取最新的任务，尝试执行（重复此过程）

```java
public static class DiscardOldestPolicy implements RejectedExecutionHandler {
    public DiscardOldestPolicy() { }

    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            e.getQueue().poll();
            e.execute(r);
        }
    }
}
```    
    

### 1.6 状态
ThreadPoolExecutor 有5个状态:

- RUNNING：接受新任务, 并且处理队列中的任务;
- SHUTDOWN：不接受新任务, 但是处理队列中的任务, 此时仍然可能创建新的线程;
- STOP：不接受新任务, 处理队列中的任务, 中断正在运行的任务;
- TIDYING：所有的任务都终结了, workCount 的值是0, 将状态转换为 TIDYING 的线程会执行 terminated() 方法;
- TERMINATED：terminated() 方法执行完毕。

状态转换:

- RUNNING -> SHUTDOWN   //调用shutdown()
- RUNNING or SHUTDOWN -> STOP //调用shutdownNow()
- SHUTDOWN -> TIDYING //queue and pool 都空了
- STOP -> TIDYING , When pool is empty
- TIDYING -> TERMINATED , When the terminated() hook method has completed


/** 
 * 1. 
 * 2. 
 * 3. 
 * 4. 
 * 5. 
 */
