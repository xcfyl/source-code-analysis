# AQS共享锁源代码分析

## 一、共享锁释放源代码

### 1.1 releaseShared

```java
@ReservedStackAccess
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

### 1.2 doReleaseShared

```java
// 只有成功释放共享锁才会执行该方法
// 该方法可能会被多个线程同时执行，因为有可能有多个线程同时拥有共享锁
private void doReleaseShared() {
  	// 进入循环
    for (;;) {
      	// 记录旧的head
        Node h = head;
      	// 下面这个if在确保两件事情
      	// aqs队列已经被正确的初始化了，即h != null
     		// 当前aqs队列中至少包含两个节点，即head.next不为null，也即h != tail
        if (h != null && h != tail) {
          	// 获取旧的head的状态值
            int ws = h.waitStatus;
          	// 如果发现旧的状态的值是SIGNAL，说明当前释放共享锁的线程需要
          	// 唤醒后继节点
            if (ws == Node.SIGNAL) {
              	// 唤醒后继节点之前，将当前头节点的状态置为中间状态0
              	// 为什么要置为中间状态0？
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                  	// 如果将头节点改为中间状态0失败，又是哪些原因会导致失败呢？
                    continue;            // loop to recheck cases
                // 如果当前释放共享锁的线程成功将head的状态改为中间状态0
              	// 那么唤醒后继节点
              	unparkSuccessor(h);
            }
          	// 如果head节点的状态不为SIGNAL，而是为0，
          	// 那么尝试将head的状态更改为PROPAGATE，那么ws什么时候会变为0呢？
            else if (ws == 0 &&
                    !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
      	// 这个条件是跳出当前循环的唯一条件
      	// h == head，意味着上述操作都做完了，head没有被改变
      	// 那么head什么时候会被改变呢？
        if (h == head)                   // loop if head changed
            break;
    }
}
```

#### 1.2.1 问题1:为什么要将head的状态置为中间状态0？

#### 1.2.2 哪些原因导致将head的状态改为0失败？

#### 1.2.3 什么时候head节点的状态不是SIGNAL而是0？

#### 1.2.4 h==head这个条件什么时候会成立？

### 1.3 unparkSuccessor

```java
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```



## 二、共享锁获取源代码

### 2.1 acquireShared

```java
public final void acquireShared(int arg) {
    // tryAcquireShared方法返回值大于0说明共享资源还有剩余，此时其他线程来获取
    // 可能依然能获取得到，如果等于0，说明当前共享资源目前已经是最后一份了，其他
    // 线程如果此时来获取很有可能就是获取不到的，需要执行doAcquireShared逻辑。
    // 如果小于0，说明当前共享资源早就被抢完了，此时线程一定会立马执行doAcquireShared
    // 方法
    if (tryAcquireShared(arg) < 0)
        // 走到这里说明共享的获取锁失败了，那么进入doAcquireShared方法
        doAcquireShared(arg);
}
```

### 2.2 doAcquireShred

```java
// 该方法在获取共享锁失败的时候调用，因为可能同时有多个线程获取共享锁失败
// 所以该方法可能会被多个线程同时执行
private void doAcquireShared(int arg) {
    // 采用循环CAS将共享类型的node添加到aqs队列的尾部
    final Node node = addWaiter(Node.SHARED);
    // 是否失败的标志位
    boolean failed = true;
    try {
        // 是否被中断的标志位
        boolean interrupted = false;
        for (; ; ) {
            // 获取当前线程的直接前驱节点
            final Node p = node.predecessor();
            if (p == head) {
                // 虽然doAcquireShared方法可能会被多个线程同时执行
                // 但是p == head这个条件在同一时刻只会有一个线程成立
                // 对于那个处于head.next位置的线程节点来说它继续尝试
                // 获取共享锁，因为很可能它在走到这里的时候，有其他线程
                // 释放了共享锁
                int r = tryAcquireShared(arg);
                // r是可能小于0的，因为有可能没有线程释放共享锁，虽然你排在
                // head之后，但是当前线程仍然获取锁失败
                if (r >= 0) {
                    // 走到这里说明获取共享锁成功
                    // 将自己设置为新的head，并做一些其他事情，具体做了什么事情，需要
                    // 具体看setHeadAndPropagate方法的分析
                    setHeadAndPropagate(node, r);
                    // 如果p指向的是一个大对象，那么在方法执行完成之前，就主动
                    // 将p引用置为null，可以帮助GC，GC不需要等待该方法执行完
                    // 就可以直接回收该对象，否则该方法在运行期间p指向的对象始
                    // 终有引用指向无法被回收
                    p.next = null; // help GC
                    // 检查是否被中断了，如果被中断了，那么需要恢复中断标志位
                    if (interrupted)
                        selfInterrupt();
                    // 走到这里，说明doAcquireShared方法没有执行失败
                    // 将其失败标志位设置为false
                    failed = false;
                    return;
                }
            }
            // 大多数的线程在获取共享锁失败后，会走到这里，在这里线程会将head的标志位
            // 设置为signal，表示当head对应的线程释放锁的时候，需要主动唤醒后继节点
            // parkAndCheckInterrupt用来将获取共享锁失败的线程挂起。一般来说，第一
            // 次执行到这里的线程都不会被挂起，而是要等待第二次走到这里的时候才会被挂起
            // 具体的原因取决于shouldParkAfterFailedAcquire方法的逻辑
            if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        // 走到这里有两种情况，要么是tryAcquireShared方法抛出异常
        // 如果是抛出异常，那么failed就会是true，走失败的处理逻辑，
        // 否则就是方法正常返回，此时failed一定为false，也就不会走
        // 失败的处理逻辑
        if (failed)
            cancelAcquire(node);
    }
}
```

### 2.3 addWaiter和enq

```java
private Node addWaiter(Node mode) {
    // 将当前线程封装到节点中，同时设置该节点的类型
    Node node = new Node(Thread.currentThread(), mode);
    // 在进入循环cas之前，先尝试一次快速的将node添加到队列的尾部，快速添加
    // 的前提是当前aqs队列至少已经被初始化了
    // 如果失败了，再进行循环cas，为了提高效率
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
```

### 2.4 enq

```java
private Node enq(final Node node) {
    for (;;) {
        // 记录当前尾节点
        Node t = tail;
        if (t == null) { // Must initialize
            // 如果发现t为空，那么尝试cas初始化tail
            // 可能会失败，因为可能同时多个线程执行该方法
            // 只有一个线程能成功
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            // 如果发现当前t不为空
            // 那么尝试将node变为新的尾巴节点
            node.prev = t;
            // 采用cas的方式将node变为新的尾巴节点
            // 该过程也可能会失败
            if (compareAndSetTail(t, node)) {
                // 执行到这里说明tail已经指向node了
                // 此时t记录的就是旧的尾节点，node的prev指针已经连接到了t
                // t的next再指向node，就把node彻底和队列连接了
                t.next = node;
                return t;
            }
        }
    }
}
```



### 2.5 setHeadAndPropagate

```java
// 该方法是在head.next位置的线程获取共享锁成功的时候会被执行
// 能执行该方法，要么是线程第一次执行doAcquireShared方法就获取
// 共享锁成功；要么是线程从park状态被唤醒，然后获取共享锁成功
private void setHeadAndPropagate(Node node, int propagate) {
  	// 记录旧的head
    Node h = head; // Record old head for check below
  	// 调用setHead，将当前节点设置为aqs队列的头节点，这里setHead并没有
  	// 使用CAS，原因是setHeadAndPropagate方法在同一时刻只可能被一个线程
  	// 调用，即处于head.next位置，并且处于活动状态的线程。
    setHead(node);
  	// propagate: 进入这个方法的时候propagate等于tryAcquireShared(arg)的返回值r
 		// r有两种可能，一种是为0，一种是>0，为0表示共享资源在此刻没有剩余了，也就是说
  	// propagate可能大于0，也可能小于0
  	// h: 能进入这个方法，肯定是先执行了addWaiter方法，因此aqs队列中至少有两个节点
  	// 所以h==null这个条件始终为false
  	// h.waitStatus: h的状态的可能取值为
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
      	// s是可能为null的，因为node可能就是整个队列中的最后一个节点
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```

