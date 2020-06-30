### 20191115
#### 说一说Handler中的IdleHandler
##### 功能

IdleHandler 可以用来提升性能，主要用在我们希望能够在当前线程 消息队列空闲时 做些事情（例如UI线程在显示完成后，如果线程空闲我们就可以提前准备其他内容）的情况下，不过最好不要做耗时操作。说白了，用的好对性能提升有帮助，用不好，一片anr，还不容易查找问题根源。

Takl is cheap. Show you the code!

首先看先源码：

```
/**
     * Callback interface for discovering when a thread is going to block
     * waiting for more messages.
     */
    public static interface IdleHandler {
        /**
         * Called when the message queue has run out of messages and will now
         * wait for more.  Return true to keep your idle handler active, false
         * to have it removed.  This may be called if there are still messages
         * pending in the queue, but they are all scheduled to be dispatched
         * after the current time.
         */
        boolean queueIdle();
    }
```

注释中明确的指出**当消息队列空闲的时候，会执行IdleHandler的queueIdle()方法，该方法返回一个false，如果为false则执行完毕之后移除这条消息，如果为true，则保留，等到下次空闲时再次执行。**看下MessageQueue的next()方法，来验证是否如所说的这样。
```
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
                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null;
                }
                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }

                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }
                 // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }

                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;

            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
            
从上面的源码，可以看到确实如上述。

处理完IdleHandler后会将nextPollTimeoutMillis设置为0，也就是不阻塞消息队列，当然要注意这里执行的代码同样不能太耗时，因为它是同步执行的，如果太耗时肯定会影响后面的message执行。

##### 适用场景

要使用IdleHandler只需要调用MessageQueue#addIdleHandler(IdleHandler handler)方法即可。

合适场景可以从以下一点或几点出发

消息队列相关
主线程能干的事情
返回true和false带来的不同结果

目前可以想到的场景

1.Activity启动优化：onCreate，onStart，onResume中耗时较短但非必要的代码可以放到IdleHandler中执行，减少启动时间

2.想要在一个View绘制完成之后添加其他依赖于这个View的View，当然这个用View#post()也能实现，区别就是前者会在消息队列空闲时执行

3.发送一个返回true的IdleHandler，在里面让某个View不停闪烁，这样当用户发呆时就可以诱导用户点击这个View，这也是种很酷的操作

4.一些第三方库中有使用，比如LeakCanary，Glide中有使用到，具体可以自行去查看
