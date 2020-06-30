# 20191216
### handler-looper源码解读，如何在msg.postDelay的情况下保证消息次序

点开postDelayed方法，通过源码我们发现，postDelayed的调用路径为：

* postDelayed(Runnable r, long delayMillis)
* sendMessageDelayed(Message msg, long delayMillis)
* sendMessageAtTime(Message msg, long uptimeMillis)
* enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis)
* enqueueMessage(Message msg, long when)

我们直捣黄龙，直接看最终的实现enqueueMessage(Message msg, long when)方法：

```
   boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```

我们知道MessageQueue中保存message的数据结构是单链表，只保留了链表头的引用。MessageQueue在进行处理的时候，只是将执行的时间设置到了message中，而没有进行任何处理......mmp。

继续看Looper中是如何读取MessageQueue的。loop()方法：

```
  /**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
    public static void loop() {

		......

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

		......
    }
```

可以看到调用的是MessageQueue的next()方法。我们看next方法中是如何实现的。

```
   Message next() {
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }
			.......
              }
    }
```

通过上面的源码，我们知道：如果头部的这个message是有延迟的，且延迟时间没到的（now<msg.when），会将该消息执行的时间与当前之间做个时间差，赋值给nextPollTimeoutMillis。然后在循环开始的时候判断如果这个Message有延迟，就调用nativePollOnce(ptr, nextPollTimeoutMillis)进行阻塞。

如果Message会阻塞MessageQueue的话，那么先postDelay10秒一个Runnable A，消息队列会一直阻塞，然后我再post一个Runnable B，B岂不是会等A执行完了再执行？正常使用时显然不是这样的，那么问题出在哪呢？

再来看MessageQueue.enqueueMessage()方法：

```

msg.markInUse();
msg.when = when;
Message p = mMessages;
boolean needWake;
if (p == null || when == 0 || when < p.when) {
    // New head, wake up the event queue if blocked.
    msg.next = p;
    mMessages = msg;
    needWake = mBlocked;
} else {
    ...
}
...
// We can assume mPtr != 0 because mQuitting is false.
if (needWake) {
    nativeWake(mPtr);

```
needWake变量和nativeWake()方法是用来唤醒线程。在接着看下next()方法：

```
Message next() {
    for (;;) {
        ...
        if (msg != null) {
            ...
        } else {
            // Got a message.
            mBlocked = false;
            ...
        }
        ...
    }
    ...
    if (pendingIdleHandlerCount <= 0) {
        // No idle handlers to run.  Loop and wait some more.
        mBlocked = true;
        continue;
    }
    ...
}
```
在next()方法中，如果有阻塞（没有消息，或者只有delay消息），则会把mBlocked设为true，在下一个message到来时就会判断这个message的位置，如果在队首就会调用nativeWake()方法唤醒线程。

以刚才的例子为例：

1. postDelay()一个10秒钟的Runnable A、消息进队，MessageQueue调用nativePollOnce()阻塞，Looper阻塞；

2. 紧接着post()一个Runnable B、消息进队，判断现在A时间还没到、正在阻塞，把B插入消息队列的头部（A的前面），然后调用nativeWake()方法唤醒线程；

3. MessageQueue.next()方法被唤醒后，重新开始读取消息链表，第一个消息B无延时，直接返回给Looper；

4. Looper处理完这个消息再次调用next()方法，MessageQueue继续读取消息链表，第二个消息A还没到时间，计算一下剩余时间（假如还剩9秒）继续调用nativePollOnce()阻塞；
5. 直到阻塞时间到或者下一次有Message进队；

这样就会保证通过postDelayed方法发送的消息会按照正确的次序传递给Looper进行处理。

#### 整理一下：

**MessageQueue会根据post delay的时间排序放入到链表中，链表头的时间小，尾部时间最大。因此能保证时间Delay最长的不会block住时间短的。当每次post message的时候会进入到MessageQueue的next()方法，会根据其delay时间和链表头的比较，如果更短则，放入链表头，并且看时间是否有delay，如果有，则block，等待时间到来唤醒执行，否则将唤醒立即执行。**
 
**所以handler.postDelay并不是先等待一定的时间再放入到MessageQueue中，而是直接进入MessageQueue，以MessageQueue的时间顺序排列和唤醒的方式结合实现的。使用后者的方式，是集中式的统一管理了所有message，而如果像前者的话，有多少个delay message，则需要起多少个定时器。前者由于有了排序，而且保存的每个message的执行时间，因此只需一个定时器按顺序next即可**
