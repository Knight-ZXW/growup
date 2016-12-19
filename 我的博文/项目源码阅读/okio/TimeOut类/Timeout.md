# Okio库 是如何处理 IO流操作

##  Timeout类的作用
　　**Timeout**类用来处理当前线程对异步任务执行的等待超时时间或截止时间，当超时或到截止时间时，抛出一个 **InterruptedIOException** 的异常，在 **Okio** 包中，处理Io流时，都需要使用到。**TimeOut** 主要包含 **Timeout** 以及**AsyncTimout** 类。

## TimeOut类内部的具体处理过程
　　**Timeout**类 使用了2种策略来处理是否应该中断等待，一种是 **任务处理的时间**，另一种是设定 **任务时间的截止点**，这两种策略也可以同时存在，判断时会取最近的临界点时间。
　　**有关这两种策略的设置方法**
```java
  /**
   * 等待任务执行完成的最长时间，如果设置为0，相当于会无限等待直到任务执行完成
   */
  public Timeout timeout(long timeout, TimeUnit unit) {
    if (timeout < 0) throw new IllegalArgumentException("timeout < 0: " + timeout);
    if (unit == null) throw new IllegalArgumentException("unit == null");
    this.timeoutNanos = unit.toNanos(timeout);
    return this;
  }
  /**
	*使用deadline 来 保证所有的任务执行应该在deadlineTime之前执行完成，否则抛出异常
   */
  public Timeout deadlineNanoTime(long deadlineNanoTime) {
    this.hasDeadline = true;
    this.deadlineNanoTime = deadlineNanoTime;
    return this;
  }

  /**  设置截止时间，内部通过调用deadlineNanoTime. */
  public final Timeout deadline(long duration, TimeUnit unit) {
    if (duration <= 0) throw new IllegalArgumentException("duration <= 0: " + duration);
    if (unit == null) throw new IllegalArgumentException("unit == null");
    return deadlineNanoTime(System.nanoTime() + unit.toNanos(duration));
  }
```

### waitUntilNotified(Object monitor)
　　该类的主要判断逻辑在 **waitUntilNotifed**，调用改方法时，传入moitor 对象，代码内部先判断 是否有设置 等待或者截止时间，接着调用 **mointor** 对象的 **wait()** 或者 带 时间参数的 **wait(long millis,int naosn)** 方法，**wait**方法的调用 导致当前线程等待直到其他线程释放monitor对象锁才有可能被唤醒。从这里也可以看出 Timeout 类 主要是对 **Object** 类  wait 方法 timout处理的封装，当持有monitor 对象锁的线程的处理时间过长时，抛出异常。
```java
  public final void waitUntilNotified(Object monitor) throws InterruptedIOException {
    try {
      boolean hasDeadline = hasDeadline();
      long timeoutNanos = timeoutNanos();

      if (!hasDeadline && timeoutNanos == 0L) {
        monitor.wait(); // There is no timeout: wait forever.
        return;
      }

      // 计算需要 wait的时间
      long waitNanos;
      long start = System.nanoTime();
      if (hasDeadline && timeoutNanos != 0) {
        long deadlineNanos = deadlineNanoTime() - start;
        waitNanos = Math.min(timeoutNanos, deadlineNanos);
      } else if (hasDeadline) {
        waitNanos = deadlineNanoTime() - start;
      } else {
        waitNanos = timeoutNanos;
      }

      // Attempt to wait that long. This will break out early if the monitor is notified.
      long elapsedNanos = 0L;
      if (waitNanos > 0L) {
        long waitMillis = waitNanos / 1000000L;
        monitor.wait(waitMillis, (int) (waitNanos - waitMillis * 1000000L));
        elapsedNanos = System.nanoTime() - start; //真实的wait 时间
      }

      // Throw if the timeout elapsed before the monitor was notified.
      if (elapsedNanos >= waitNanos) {
        throw new InterruptedIOException("timeout");
      }
    } catch (InterruptedException e) {
      throw new InterruptedIOException("interrupted");
    }
  }
```

## AsyncTimeout

　　**AsyncTimeout**  类继承于 **Timeout** ，内部关于判断操作是否超时的逻辑与基类一样，不同之处在于，Timeout的判断处理是异步的，并且 AsyncTimeout 自身 采用了双向链表（**Queue**）的结构排序，内部使用了一个线程来处理链表，判断是否超时。**Okio** 主要在对 **Socket** 读写时使用到 **AsyncTimout** 类，
### 具体实现
　　创建的AsyncTimeout 在工作之前 需要调用 **enter()** 函数 。
```java
  public final void enter() {
    if (inQueue) throw new IllegalStateException("Unbalanced enter/exit");
    long timeoutNanos = timeoutNanos();
    boolean hasDeadline = hasDeadline();
    if (timeoutNanos == 0 && !hasDeadline) {
      return; // No timeout and no deadline? Don't bother with the queue.
    }
    inQueue = true;
    scheduleTimeout(this, timeoutNanos, hasDeadline);
  }
 private static synchronized void scheduleTimeout(
      AsyncTimeout node, long timeoutNanos, boolean hasDeadline) {
    // Start the watchdog thread and create the head node when the first timeout is scheduled.
    if (head == null) {
      head = new AsyncTimeout();
      new Watchdog().start();
    }

    long now = System.nanoTime();
    if (timeoutNanos != 0 && hasDeadline) {
      // Compute the earliest event; either timeout or deadline. Because nanoTime can wrap around,
      // Math.min() is undefined for absolute values, but meaningful for relative ones.
      node.timeoutAt = now + Math.min(timeoutNanos, node.deadlineNanoTime() - now);
    } else if (timeoutNanos != 0) {
      node.timeoutAt = now + timeoutNanos;
    } else if (hasDeadline) {
      node.timeoutAt = node.deadlineNanoTime();
    } else {
      throw new AssertionError();
    }

    // Insert the node in sorted order.
    long remainingNanos = node.remainingNanos(now);
    for (AsyncTimeout prev = head; true; prev = prev.next) {
      if (prev.next == null || remainingNanos < prev.next.remainingNanos(now)) {
        node.next = prev.next;
        prev.next = node;
        if (prev == head) {
          AsyncTimeout.class.notify(); // Wake up the watchdog when inserting at the front.
        }
        break;
      }
    }
  }
```


　　在 **enter()** 函数内部 首先做了 timeoutNaos 以及 hasDeadline 参数的 合法性检验，之后对 整个队列进行遍历，将这个新的 **AsyncTimout** 插入队列中。在插入队列之前，还判断了 **AsyncTimeout** 静态成员变量 **head**(作为链表头的引用) 是否存在，如果不存在，则说明该类是第一次被实例化，则创建一个 该类 唯一的守护线程 **watchDog**，**watchDog** 内部所做的逻辑也非常简单，通过一个死循环 内部 判断 队列首部距离超时的时间，之后 调用 **wait(long millis, int nanos)** ,将watchDog线程挂起对应的时间，之后进入下一次循环，再次判断是否超时，如无意外，此时队首的 **AsyncTimeout** 已经超时，调用 子类重载的 **timeOut** 方法。
　　这里需要注意，使用者应该 重载 AsyncTimeout 的timeOut方法，以实现在超时时需要做的逻辑操作，比如 关闭 IO流。

```java
/**
* Watchdog 线程处理 AsyncTimeout队列。当超时时调用 AsyncTimeout的 timeout回调方法。 
*/
private static final class Watchdog extends Thread {
    public Watchdog() {
      super("Okio Watchdog");
      setDaemon(true);
    }

    public void run() {
      while (true) {
        try {
          AsyncTimeout timedOut = awaitTimeout(); // 判断队首的超时时间，无超时则wait对应时间返回null,否则返回超时队首TimeOut

          // Didn't find a node to interrupt. Try again.
          if (timedOut == null) continue;

          // Close the timed out node.
          timedOut.timedOut();
        } catch (InterruptedException ignored) {
        }
      }
    }
  }
```
　　在　**AsyncTimout** 所绑定的工作内容完成时，调用者 调用**exit()** 方法，将 **AsyncTimeout** 从队列中移除。 由于 **timeOut** 这个重载方法的回调是异步的，所以这个方法可能在 **exit()** 方法之后被调用。
```java
 /** Returns true if the timeout occurred. */
  public final boolean exit() {
    if (!inQueue) return false;
    inQueue = false;
    return cancelScheduledTimeout(this);
  }

  /** Returns true if the timeout occurred. */
  private static synchronized boolean cancelScheduledTimeout(AsyncTimeout node) {
    // Remove the node from the linked list.
    for (AsyncTimeout prev = head; prev != null; prev = prev.next) {
      if (prev.next == node) {
        prev.next = node.next;
        node.next = null;
        return false;
      }
    }

    // The node wasn't found in the linked list: it must have timed out!
    return true;
  }
```
