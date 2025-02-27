---
draft: false
date: 2025-02-27T15:52:52+08:00
title: "Java实现生产者消费者模型"
slug: "java-producer-consumer-model" 
tags: ["java","juc]
categories: []
authors: ["since"]
description: ""
disableShare: true # 底部不显示分享栏
---

## 案例

Java并发的时候，看到了一个关于生产者消费者的案例

一个producer类和一个consumer类，实现了Runnable接口，作为线程执行的任务



一个Queue类，实现了queue的put()和take操作，模拟了生产和消费2个动作



实现方式使用了ReentrantLock+Condition



这个示例代码的话，是可以实现多线程之间并发生产和消费的，多个线程共同操作一个Queue对象，可以实现队列满时生产等待，队列空时消费等待，且无并发问题。



**我当时产生的疑问是为什么使用了ReentrantLock，还要使用Condition**



这个问题下一个章节再说，先贴下示例代码





Producer

```java
public class Producer implements Runnable {

    Queue queue = null;

    public Producer(Queue queue) {
        this.queue = queue;
    }

    @Override
    public void run() {
        String threadName = Thread.currentThread().getName();

        try {
            // 隔10秒轮询生产一次
            while (true) {
                System.out.println("Producer");
                TimeUnit.SECONDS.sleep(10);
                queue.put(new Random().nextInt(100),threadName);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```



Consumer

```java
public class Consumer implements Runnable {
    Queue queue = null;

    public Consumer(Queue queue) {
        this.queue = queue;
    }

    @Override
    public void run() {
        String threadName = Thread.currentThread().getName();
        try {

            // 隔3秒轮询消费一次
            while (true) {
                System.out.println("Customer");
                TimeUnit.SECONDS.sleep(3);
                System.out.println("取到的值-" + queue.take());
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```



Queue

```java
public class Queue {
    private int[] arr = new int[5];
    private int size = 0;
    private int putIndex = 0;  // 生产位置
    private int takeIndex = 0; // 消费位置
    private ReentrantLock lock = new ReentrantLock();
    private Condition pCondition = lock.newCondition();
    private Condition cCondition = lock.newCondition();

    public boolean isEmpty() {
        return size==0;
    }

    public boolean isFull() {
        return size==5;
    }

    public void put(Integer value, String name) throws InterruptedException {
        log(name + " ▶▶▶ 尝试获取锁...");
        lock.lock();
        try {
            log(name + " ✅ 成功获取锁 | 当前锁状态: " + lock);
            while (isFull()) {
                log(name + " ⏸️ 队列已满，进入等待 (pCondition.await())...");
                pCondition.await(); // 自动释放锁！
                log(name + " 🔄 被唤醒，重新获取锁 | 当前锁状态: " + lock);
            }
            arr[putIndex] = value;
            putIndex = (putIndex + 1) % arr.length;
            size++;
            // 生产逻辑...
            log(name + " 🔔 生产完成，唤醒消费者 (cCondition.signalAll())");
            cCondition.signalAll();
        } finally {
            lock.unlock();
            log(name + " ⏹️ 释放锁 | 当前锁状态: " + lock);
        }
    }

    public int take() throws InterruptedException {
        String threadName = Thread.currentThread().getName();
        log(threadName + " ▶▶▶ 尝试获取锁...");
        lock.lock();
        try {
            log(threadName + " ✅ 成功获取锁 | 当前锁状态: " + lock);
            while (isEmpty()) {
                log(threadName + " ⏸️ 队列为空，进入等待 (cCondition.await())...");
                cCondition.await(); // 自动释放锁！
                log(threadName + " 🔄 被唤醒，重新获取锁 | 当前锁状态: " + lock);
            }
            int value = arr[takeIndex];
            arr[takeIndex] = 0;  // 可选：清理数据以便调试
            takeIndex = (takeIndex + 1) % arr.length;
            size--;
            log(threadName + " 🔔 消费完成，唤醒生产者 (pCondition.signalAll())");
            pCondition.signalAll();
            return value;
        } finally {
            lock.unlock();
            log(threadName + " ⏹️ 释放锁 | 当前锁状态: " + lock);
        }
    }

    private void log(String message) {
        System.out.printf("[%s] %s%n", LocalDateTime.now().format(DateTimeFormatter.ISO_LOCAL_TIME), message);
    }
}
```



## 分析

### 代码测试

首先写了个测试类，测试了一下生产消费逻辑, 2个生产者，2个消费者，共4个线程，操作同一个queue，实现生产消费逻辑

```java
public static void main(String[] args) {
        // 两个生产者，两个消费者
        Queue queue = new Queue();
        Thread producer1 = new Thread(new Producer(queue));
        producer1.setName("Producer1");
        producer1.start();
        Thread producer2 = new Thread(new Producer(queue));
        producer2.setName("Producer2");
        producer2.start();
        Thread Consumer1 = new Thread(new Consumer(queue));
        Consumer1.setName("Consumer1");
        Consumer1.start();

        Thread Consumer2 = new Thread(new Consumer(queue));
        Consumer2.setName("Consumer2");
        Consumer2.start();
}
```





控制台部分输出内容如下

输出内容展示了尝试获取锁---->获取锁成功---->等待---->释放锁---->被唤醒---->重新获取锁---->生产/消费完成并唤醒等待的线程---->释放锁的这么一个过程

```java
[16:14:07.504] Consumer1 ▶▶▶ 尝试获取锁...
[16:14:07.504] Consumer2 ▶▶▶ 尝试获取锁...
[16:14:07.516] Consumer1 ✅ 成功获取锁 | 当前锁状态: java.util.concurrent.locks.ReentrantLock@363b2ca2[Locked by thread Consumer1]
[16:14:07.516] Consumer1 ⏸️ 队列为空，进入等待 (cCondition.await())...
[16:14:07.516] Consumer2 ✅ 成功获取锁 | 当前锁状态: java.util.concurrent.locks.ReentrantLock@363b2ca2[Locked by thread Consumer2]
[16:14:07.516] Consumer2 ⏸️ 队列为空，进入等待 (cCondition.await())...
[16:14:14.5] Producer2 ▶▶▶ 尝试获取锁...
[16:14:14.501] Producer2 ✅ 成功获取锁 | 当前锁状态: java.util.concurrent.locks.ReentrantLock@363b2ca2[Locked by thread Producer2]
[16:14:14.502] Producer2 🔔 生产完成，唤醒消费者 (cCondition.signalAll())
[16:14:14.5] Producer1 ▶▶▶ 尝试获取锁...
[16:14:14.502] Consumer1 🔄 被唤醒，重新获取锁 | 当前锁状态: java.util.concurrent.locks.ReentrantLock@363b2ca2[Locked by thread Consumer1]
[16:14:14.502] Producer2 ⏹️ 释放锁 | 当前锁状态: java.util.concurrent.locks.ReentrantLock@363b2ca2[Unlocked]
Producer
[16:14:14.503] Consumer1 🔔 消费完成，唤醒生产者 (pCondition.signalAll())
[16:14:14.504] Consumer2 🔄 被唤醒，重新获取锁 | 当前锁状态: java.util.concurrent.locks.ReentrantLock@363b2ca2[Locked by thread Consumer2]
[16:14:14.504] Consumer1 ⏹️ 释放锁 | 当前锁状态: java.util.concurrent.locks.ReentrantLock@363b2ca2[Unlocked]
[16:14:14.504] Consumer2 ⏸️ 队列为空，进入等待 (cCondition.await())...
取到的值-68
Customer
[16:14:14.504] Producer1 ✅ 成功获取锁 | 当前锁状态: java.util.concurrent.locks.ReentrantLock@363b2ca2[Locked by thread Producer1]
[16:14:14.505] Producer1 🔔 生产完成，唤醒消费者 (cCondition.signalAll())
[16:14:14.505] Producer1 ⏹️ 释放锁 | 当前锁状态: java.util.concurrent.locks.ReentrantLock@363b2ca2[Unlocked]
Producer
[16:14:14.505] Consumer2 🔄 被唤醒，重新获取锁 | 当前锁状态: java.util.concurrent.locks.ReentrantLock@363b2ca2[Locked by thread Consumer2]
[16:14:14.506] Consumer2 🔔 消费完成，唤醒生产者 (pCondition.signalAll())
[16:14:14.506] Consumer2 ⏹️ 释放锁 | 当前锁状态: java.util.concurrent.locks.ReentrantLock@363b2ca2[Unlocked]
取到的值-83
```



### 疑问

一开始由于对这快内容不是很了解，我产生了2个疑问，

- 线程只要获取锁之后，其他线程都得等待获取锁，多个线程只用竞争一把锁，为什么要使用condition呢。
- 线程在获取锁之后，只能等待锁执行完释放才能重新竞争到锁，加了condition等待唤醒有什么用呢



带着这2个疑问，我去问了deepseek和grok以及chatgpt，靠ai给思路，然后去看了代码，确认了使用condition的合理性和必要性



#### 问题1

先看看grok怎么回答问题1的



**只使用lock**

单独使用 Lock 通常会通过忙等待（busy-waiting）或简单的条件检查来模拟生产者-消费者的同步

```java
Lock lock = new ReentrantLock();
Queue<Integer> queue = new LinkedList<>();
int capacity = 10;

public void put(int item) {
    lock.lock();
    try {
        while (queue.size() >= capacity) {
            lock.unlock(); // 临时释放锁
            Thread.yield(); // 让出 CPU
            lock.lock();   // 重新获取锁
        }
        queue.add(item);
    } finally {
        lock.unlock();
    }
}

public void take() {
    lock.lock();
    try {
        while (queue.isEmpty()) {
            lock.unlock();
            Thread.yield();
            lock.lock();
        }
        queue.poll();
    } finally {
        lock.unlock();
    }
}
```



只使用lock，会频繁判断条件，写出如上的加锁/释放锁的代码，以及要让出cpu，不够优雅



**结合lock+condition**

使用 Lock 和 Condition，可以通过条件变量精确控制线程的等待和唤醒

```java
Lock lock = new ReentrantLock();
Condition pCondition = lock.newCondition();
Condition cCondition = lock.newCondition();
Queue<Integer> queue = new LinkedList<>();
int capacity = 10;

public void produce(int item) {
    lock.lock();
    try {
        while (queue.size() >= capacity) {
            pCondition.await(); // 等待“非满”条件
        }
        queue.add(item);
        cCondition.signal(); // 唤醒等待“非空”的消费者
    } finally {
        lock.unlock();
    }
}

public void consume() {
    lock.lock();
    try {
        while (queue.isEmpty()) {
            cCondition.await(); // 等待“非空”条件
        }
        queue.poll();
        pCondition.signal(); // 唤醒等待“非满”的生产者
    } finally {
        lock.unlock();
    }
}
```



lock+condition，可以精准控制线程的等待和唤醒

使用了2个condition，queue满了pCondition就等待，暂停生产，queue空了cCondition就等待，暂停消费

生产之后， 释放信号，唤醒cCondition去消费，消费之后，唤醒pCondition去生产





**2种实现方式优缺点比较**

| 特性           | 只使用 Lock                | 使用 Lock + Condition        |
| -------------- | -------------------------- | ---------------------------- |
| **线程通知**   | 无法精确通知，依赖轮询     | 精确通知，通过 signal() 唤醒 |
| **效率**       | 忙等待或频繁锁切换，效率低 | 线程挂起，无忙等待，效率高   |
| **锁竞争**     | 高，可能反复释放和获取锁   | 低，唤醒后队列式获取锁       |
| **代码复杂度** | 高，手动实现等待逻辑       | 低，API 简洁且功能强大       |
| **条件区分**   | 无法区分不同条件           | 支持多个 Condition 对象      |
| **中断处理**   | 手动检查和处理             | 内置支持，抛出异常           |



通过以上的优缺点比较，可以看到单独使用lock，有以下缺点

- 缺乏精确的线程通知机制
- 效率低下，忙等待或锁切换开销大。
- 更高的锁竞争和延迟。
- 代码复杂且不够优雅。
- 无法区分不同等待条件。
- 中断处理困难。





在实际的生产场景种

假设一个高并发生产者-消费者系统：

- 只使用 Lock
  - 缓冲区满时，生产者线程不断轮询，浪费 CPU。
  - 消费者被唤醒后可能发现缓冲区仍空（其他消费者抢先消费），需要再次等待。
  - 系统吞吐量低，延迟高。
- 使用 Lock + Condition
  - 生产者等待 pCondition，消费者等待 cCondition，互不干扰。
  - 生产者放入数据后只唤醒消费者，消费者移除数据后只唤醒生产者。
  - 系统高效运行，资源利用率高。



#### 问题2

再看看问题2的回答

获取condition使用的是lock的api，返回的是定义在aqs中的conditionobject

```java
private ReentrantLock lock = new ReentrantLock();
private Condition pCondition = lock.newCondition();
private Condition cCondition = lock.newCondition();
```

![image-20250227172213797](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202502271722889.png)



调用`pCondition.await(); `实际是调用了`java.util.concurrent.locks.AbstractQueuedSynchronizer.ConditionObject#await()`

我将aqs的方法代码，喂给了grok，让grok帮我分析方法实现的细节

```java
/**
 * 实现可中断的条件等待。
 * <ol>
 * <li> 如果当前线程被中断，则抛出 InterruptedException 异常。
 * <li> 保存由 {@link #getState} 返回的锁状态。
 * <li> 使用保存的状态作为参数调用 {@link #release} 方法，
 *      如果失败则抛出 IllegalMonitorStateException 异常。
 * <li> 阻塞，直到被唤醒或被中断。
 * <li> 通过调用 {@link #acquire} 的特定版本，并传入保存的状态作为参数来重新获取锁。
 * <li> 如果在第 4 步阻塞时被中断，则抛出 InterruptedException 异常。
 * </ol>
 */
  

        /**
         * Implements interruptible condition wait.
         * <ol>
         * <li> If current thread is interrupted, throw InterruptedException.
         * <li> Save lock state returned by {@link #getState}.
         * <li> Invoke {@link #release} with saved state as argument,
         *      throwing IllegalMonitorStateException if it fails.
         * <li> Block until signalled or interrupted.
         * <li> Reacquire by invoking specialized version of
         *      {@link #acquire} with saved state as argument.
         * <li> If interrupted while blocked in step 4, throw InterruptedException.
         * </ol>
         */
        public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
           //释放锁
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
```



grok帮我总结出了如下重点

```java
锁的释放：
在 fullyRelease(node) 中释放锁。
发生在等待之前，确保线程在挂起时不持有锁。
等待锁的地方：
在 while (!isOnSyncQueue(node)) 循环中，通过 LockSupport.park(this) 挂起线程。
等待条件是 node 被转移到同步队列（由 signal() 触发）或被中断。
获取锁后的恢复：
在 acquireQueued(node, savedState) 中重新竞争并获取锁。
使用保存的 savedState 恢复锁的状态（例如重入次数）。
获取锁后，处理可能的清理和中断逻辑。


锁的流程如下
  
持有锁
  ↓
检查中断 → 中断抛异常退出
  ↓
创建等待节点
  ↓
释放锁 (fullyRelease)
  ↓
循环等待 (LockSupport.park)
  ↓
被唤醒/中断 → 检查中断状态
  ↓
重新获取锁 (acquireQueued)
  ↓
清理取消节点
  ↓
处理中断
  ↓
返回 (持有锁)
```



也就是说在这行代码`   int savedState = fullyRelease(node);`会释放掉锁

详细看一下这个方法，这个方法是aqs提供的`java.util.concurrent.locks.AbstractQueuedSynchronizer#fullyRelease`

```java
/**

* 使用当前状态值调用 release 方法，并返回保存的状态。
* 若操作失败，则取消节点并抛出异常。
* @param node 等待中的条件节点
* @return 先前的同步状态
*/

/**
     * Invokes release with current state value; returns saved state.
     * Cancels node and throws exception on failure.
     * @param node the condition node for this wait
     * @return previous sync state
     */
    final int fullyRelease(Node node) {
        boolean failed = true;
        try {
            int savedState = getState();
            if (release(savedState)) {
                failed = false;
                return savedState;
            } else {
                throw new IllegalMonitorStateException();
            }
        } finally {
            if (failed)
                node.waitStatus = Node.CANCELLED;
        }
    }
```



会在if条件里执行`java.util.concurrent.locks.AbstractQueuedSynchronizer#release`方法，尝试释放锁

```java
/**  
 * 以独占模式释放锁。若 {@link #tryRelease} 返回 true，则唤醒一个或多个线程。  
 * 此方法可用于实现 {@link Lock#unlock}。  
 *  
 * @param arg 释放参数，该值会传递给 {@link #tryRelease}，但不会被额外解释，可用于表示任意含义。  
 * @return {@link #tryRelease} 方法的返回值。  
 */


/**
     * Releases in exclusive mode.  Implemented by unblocking one or
     * more threads if {@link #tryRelease} returns true.
     * This method can be used to implement method {@link Lock#unlock}.
     *
     * @param arg the release argument.  This value is conveyed to
     *        {@link #tryRelease} but is otherwise uninterpreted and
     *        can represent anything you like.
     * @return the value returned from {@link #tryRelease}
     */
    @ReservedStackAccess
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```



最终执行`java.util.concurrent.locks.AbstractQueuedSynchronizer#tryRelease`进行释放，由子类定义具体的释放逻辑，这个场景下最终调用`java.util.concurrent.locks.ReentrantReadWriteLock.Sync#tryRelease`进行的释放逻辑



这么看下来，执行`java.util.concurrent.locks.Condition#await()`会将持有的锁释放掉，释放掉锁之后，其他等待获取锁的线程可以尝试获取锁，拿到锁之后尝试进行生产或者消费



假如执行到`java.util.concurrent.locks.Condition#signalAll`，那么会发通知等待在pCondition或cCondition上的线程，唤醒等待，尝试获取锁，在put()或take()finally里，最终会释放锁，对应的condition被唤醒，尝试重新获取锁。



只有这样在await会释放锁，lock+condition的组合才会有意义。



### 总结

lock+condition结合，可以实现精准控制，准确通知线程，降低锁竞争和资源消耗，实现并发的生产消费模型



## 其他

以前有问题，只能靠搜索引擎给答案，现在有了ai之后，可以向ai提问，ai可以给你做详细的解释，也不用自己苦哈哈的看和理解了。



遇到不懂的，ai能给你做详细解释，开发的学习成本确实降低了



但是ai浪潮席卷下，硅基如何干碎碳基，可能很快就会来临了。



现在就是ai人类化，人类ai化，真的抽象



