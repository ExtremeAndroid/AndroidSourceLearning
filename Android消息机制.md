
## Android消息机制

### 1. 概述

#### Handler使用示例：

```java
//handleMessage 在创建handler的线程中执行
private Handler handler = new Handler(){
    @Override
    public void handleMessage(Message msg) {
        super.handleMessage(msg);
        switch (msg.what) {
            case 1:
                break;
        }
    }
};

public class WorkThread extends Thread {
    @Override
    public void run() {
        super.run();
         // code ... 耗时操作
        Message msg =Message.obtain();
        msg.obj = data;
        msg.what=1;
        handler.sendMessage(msg);
    }
}

new WorkThread().start();
```

#### Handler工作过程

1. Handler创建完成后，其内部的Looper和MessageQueue开始协同工作。
2. Handler#post(Runnable)投递一个Runnable到Handler内部的Looper中去处理，也可以通过Handler#send()发送一个msg，同样会到Looper中去处理。
3. send()方法被调用后，他调用MessageQueue#enqueueMessage()放入消息队列。
4. 当Looper发现有新消息到来时，会处理此消息。
5. 最终，消息中的Runnable或者Handler中的handleMessage会调用来处理消息。
6. 注意，Looper是运行在创建Handler的线程中，这样一来，Handler中的业务就被切换到创建Handler所在的线程中去执行。

<img src="https://github.com/ExtremeAndroid/image/blob/master/Android%E6%B6%88%E6%81%AF%E6%9C%BA%E5%88%B6.jpg?raw=true" width=50% height=50% />
<img src="https://upload-images.jianshu.io/upload_images/1289721-b355ff5279721171.png?imageMogr2/auto-orient/" width=50% height=50% />


### 2. ThreadLocal
线程内部的数据存储，其他线程无法访问。
在Handler中负责保存每个线程各自的Looper

#### 2.1 使用场景

1. 不同线程有不同数据的时候。

	例如：Handler需要获取当前线程的Looper，显然每个线程有自己的Looper。可以用ThreadLocal来保存。
	
	否则，系统需要维护一张全局的哈希表，保存所有的Lopper，并且提供一个LooperManager访问使用。
	
2. 复杂逻辑下的对象传递。

	例如：某线程中任务复杂，调用栈深，但是又需要监听任务执行的状态。采用ThreadLocal，可以ThreadLocal持有监听器，则监听器是线程内全局的变量，调用get则可以获得监听器对象。
	
	否则，可以采用： 
	
	- 将监听器作为参数传递 --> 过渡传递，程序设计糟糕
	- 在线程外设置静态全局监听器 --> 不具有可扩展性。多有20个线程，则需要有20个全局监听器。并且封装性也不好，最好自包含。
    
#### 2.2 常用方法

```	java
// ThreadLocalMap: hash map，ThreadLocal的数据保存在此map中
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

### 3. MessageQueue
消息队列的具体实现，包含入队、出队

```java
/** 插入队列
 * 1. 单链表存取方便，用来实现消息队列
 * 2. 加锁存取
 * 3. 若标记退出，则不入队列
 */
boolean enqueueMessage(Message msg, long when) {
    synchronized (this) {
        if (mQuitting) {	//标记为退出，禁止入队
            return false;
        }
		// code ... 链表插入操作，已省略
    }
    return true;
}


/** 读取队列
 * 1. 无限循环，若没有消息，则阻塞等待
 * 2. 多有消息，则next返回此消息，并从链表删除。
 */
Message next() {
    for (;;) {
        synchronized (this) {
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
            		// 寻找下一条msg，会被阻塞。
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {// 下一条msg还没到执行时间，定个闹钟，到点后执行。
                    // code ... 
                } else { 				//可执行的msg
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    msg.markInUse();
                    return msg;		// 返回msg。
                }
            }
            
            // 检查是否退出，return null 会导致loop退出
            if (mQuitting) {
                dispose();
                return null;
            }
        }
    }
}

/** 退出Queue
 * 1. Looper调用quit后，会调用此方法，清空队列。
 * 2. mQuitAllowed只 有且只有 主线程设置为false，所以可以直接抛出异常“主线程不允许退出”。
 */
void quit(boolean safe) {
    if (!mQuitAllowed) {
        throw new IllegalStateException("Main thread not allowed to quit.");
    }
	// code ... 清空queue，省略
}
```
### 4. Looper
负责loop的执行，包含准备、执行、结束三部分

```java
/** 获取主线程的Looper
 * 1. 实际上设计层面不能保证只允许主线程来访问，所以在代码实现上看起来比较脏。具体实现如下：
 * 2. 方法名标注主线程的Looper
 * 3. 注释写Android环境主线程会自动调用此方法，用户不能调用
 * 4. 初始化的主线程ActivityThread的时候，直接调用此方法，并保存实例sMainLooper
 * 5. 其他方法若调用，则判断sMainLooper==null？，若不为null则表明主线程已启用，此次调用非法，抛出“主线程已启用的”异常。
 */
public static void prepareMainLooper() {
    prepare(false);
    synchronized (Looper.class) {
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}

```
Looper执行顺序为：prepare准备-->loop启动循环-->quit退出looper

```java
/** Looper准备
 * 1. prepare()默认调用prepare(true)，允许手动调用quit()取消。
 * 2. prepareMainLooper()调用prepare(false)，主线程不允许退出。
 * 3. prepare(boolean quitAllowed)为private，只有prepareMainLooper()传入false。外界只能调用prepare()，即实现prepare(true)。
 */
public static void prepare() {
    prepare(true);
}
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}

/** 运行当前线程中的MessageQueue
 * 1. 调用loop()之后，必须调用quit()退出。
 * 2. loop()是一个死循环，只有MessageQueue#next()返回null，才会退出。
 * 3. Looper#quit()调用之后，会调用MessageQueue#quit()/quitSafely()方法，通知消息队列退出，标记退出状态。
 * 4. MessageQueue标记退出状态后，next()方法会返回null。此时，loop死循环才能解除。
 * 5. loop()中调用MessageQueue#next()方法获取新消息。若队列没有消息，则next()阻塞，导致loop阻塞等待消息。
 * 6. 若 MessageQueue#next()返回消息，loop中接下来执行msg.target.dispatchMessage(msg)。
 * 7. msg.target指的是发送此消息的Handler对象，这样，Handler发送的消息，最终交给自己的dispatchMessage(msg)来处理。
 * 8. 但是， dispatchMessage运行在创建Handler时候的Looper中，则将代码执行逻辑切换到对应的线程中去执行。
 */
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    for (;;) {
        Message msg = queue.next();	// might block
        if (msg == null) {
            return;						// No message indicates that the message queue is quitting.
        }
        try {
            msg.target.dispatchMessage(msg);
        } finally {
        }
        msg.recycleUnchecked();
    }
}

/** 退出Looper
 * 1. quit()标记退出标志，直接清除所有消息，退出Looper。
 * 2. quitSafely()标记退出标记，不再接受新的message，等待MessageQueue中消息执行完毕后，退出Looper。
 * 3. 标记退出标记，不再接受新的message，例如：Handler#sendMessage(Message)会返回false
 * 4. mQueue.quit，具体实现见MessageQueue#quit(boolean)
 */
public void quit() {
    mQueue.quit(false);
}
public void quitSafely() {
    mQueue.quit(true);
}
```

### 5. Handler
与用户打交道的class，主要工作包含发送消息、接收消息

```java
/** 初始化
 * 1. 会检测looper是否创建，线程中必须先创建looper，才能运行Handler
 */
public Handler(Callback callback, boolean async) {
	// code ...
	mLooper = Looper.myLooper();
	if (mLooper == null) {
	    throw new RuntimeException(
	        "Can't create handler inside thread " + Thread.currentThread()
	                + " that has not called Looper.prepare()");
	}
	// code ...
}

/** 发送消息
 * 1. Handler# post()、sendMessage()、sendMessageDelayed()、sendMessageAtTime()属于入口不同，实质相同;
 * 2. 最终调用 queue.enqueueMessage(msg, uptimeMillis);
 * 3. 返回值true表示插入队列成功，不代表一定能执行;
 * 4. MessageQueue#enqueueMessage(msg)后，next()会执行，return此msg;
 * 5. Looper#loop接收到此msg，调用Handler#dispatchMessage(msg),进入处理消息阶段.
 */

public final boolean post(Runnable r){
   return  sendMessageDelayed(getPostMessage(r), 0);
}

public final boolean sendMessage(Message msg){
    return sendMessageDelayed(msg, 0);
}

public final boolean sendMessageDelayed(Message msg, long delayMillis){
	return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}
    
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
	return enqueueMessage(mQueue, msg, uptimeMillis);
}
    
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
	msg.target = this;
	if (mAsynchronous) {
	    msg.setAsynchronous(true);
	}
	return queue.enqueueMessage(msg, uptimeMillis);
}


/** 执行队列
 * 1. 检测分发，类似于责任模式。
 * 2. msg.callback，callback是Handler#post(Runnable)传进来的runnbale。
 * 3. mCallback 是直接实例化Handler产生的，若实例化子类，则不会产生此callback
 * 4. handleMessage(),子类必须实现此方法去处理msg。
 */
 public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
public void handleMessage(Message msg) {}
```

### 6. ActivityThread（主线程）的消息循环
主程序入口为main(),系统会创建Looper、MessageQueue、并启动。子线程需要自己创建。

```java
public static void main(String[] args) {
	// code ...
	Looper.prepareMainLooper();
	if (sMainThreadHandler == null) {
	    sMainThreadHandler = thread.getHandler();
	}
	Looper.loop();
	// code ...
}
```

#### 为什么要在主线程访问UI？
主线程不是线程安全的线程，若要线程安全，就需要加锁机制保证。

但是，加锁会造成访问麻烦：

1. UI访问机制复杂； 
2. 降低UI线程效率。
