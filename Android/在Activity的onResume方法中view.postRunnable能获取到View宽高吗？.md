## 20191206
### 在Activity的onResume方法中view.postRunnable能获取到View宽高吗？

可以获取。

#### 源码分析

先看View.post()方法的源码：

```
public boolean post(Runnable action) {
        final AttachInfo attachInfo = mAttachInfo;//A set of information given to a view when it is attached to its parent window.
        if (attachInfo != null) {
            return attachInfo.mHandler.post(action);
        }

        // Postpone the runnable until we know on which thread it needs to run.
        // Assume that the runnable will be successfully placed after attach.
        getRunQueue().post(action);
        return true;
    }
```

可以看到如果attachInfo不为空是，会调用**AttachInfo**的mHadler.post方法，而当attachInfo为空的时候，会调用getRunQueue().post()方法。这个方法是干啥的？看注释我们可以知道：

**将runnable延迟到我们知道它需要在哪个线程上运行。**

**假定可运行程序将在附加之后成功地放置。**

**说成大白话就是这个runnable会延迟到这个runnable被成功的attch后执行**

我们知道当onCreate方法执行的时候，view还没有attched,所以此时的attachInfo为null。那么先让我们看下getRunQueue()方法。

```
private HandlerActionQueue getRunQueue() {
        if (mRunQueue == null) {
            mRunQueue = new HandlerActionQueue();
        }
        return mRunQueue;
    }
```

嗯。。。很简单，就四行代码，返回了一个HandlerActionQueue类型的mRunQueue对象。我们来看下mRunQueue是干嘛的。

```
 Queue of pending runnables. Used to postpone calls to post() until this view is attached and has a handler.
```
就是说，view在attch之前**（dispatchAttachedToWindow被调用之前)**，mRunQueue中的runnable都会被挂起，直到这个view attch之后。

那么问题来了，啥时候attch呢？

在ViewRootImpl的performTraversals方法第一次被调用的时候。

```
private void performTraversals() {
        // cache mView since it is used so much below...
        final View host = mView;
        //注：该方法有800行代码，已省略
        // Execute enqueued actions on every traversal in case a detached view enqueued an action
        getRunQueue().executeActions(attachInfo.mHandler);

    ...
    performMeasure();//从DecorView开始完成View树的测量
    ...
    performLayout();//从DecorView开始完成View树的布局
    ...
    performDraw();//从DecorView开始绘制View树
 }
```

那么有同学就会问了，**我也看了源码，知道了RootView的layout方法是在ViewRootImpl的performLayout方法里面调用的，但是我明明看到，dispatchAttachedToWindow方法是在performLayout方法之前调用的！！！那不就是说，dispatchAttachedToWindow回调的时候，View还没layout嘛？那怎么能获取到的？**

确实，dispatchAttachedToWindow方法回调的时候，view还没有layout。

但是不要忘记了，view.post进入的runnable是会延迟执行的，所以主线程一定会等待performTraversals方法执行完再去执行，从上面的源码我们看到，在这个方法中，performMeasure, performLayout, performDraw都会执行，也就是说当整个View树完成了测量、布局、绘制之后，才回去执行我们post进去的runnable，这个时候当然可以获取到view的宽高了！


## 坑

据说在android 7.0 api24以上，这种方式已经不能100%执行。

在Android 7.0 api24，Android 8.0 api25的手机上如果通过new创建的View，如果没有将它通过addView()加入到ViewGroup布局中，那通过View.post()发送出去的任务将不再执行，也就无法通过Viwe.post更新UI。

回顾上文，记得在执行getRunQueue方法的时候，有个注释：

```
 // Assume that the runnable will be successfully placed after attach.
```
 
 假设会成功执行。。。。那就是会不执行。（坑啊）
 
具体是怎么不执行呢？

先来看下最终处理任务的方法：

```
public void executeActions(Handler handler) {
        synchronized (this) {
            final HandlerAction[] actions = mActions;
            for (int i = 0, count = mCount; i < count; i++) {
                final HandlerAction handlerAction = actions[i];
                handler.postDelayed(handlerAction.action, handlerAction.delay);
            }

            mActions = null;
            mCount = 0;
        }
    }
```

这个方法哪里调用了呢？没错，dispatchAttachedToWindow方法。

```
    void dispatchAttachedToWindow(AttachInfo info, int visibility) {
......
// Transfer all pending runnables.
        if (mRunQueue != null) {
            mRunQueue.executeActions(info.mHandler);
            mRunQueue = null;
        }
 .....

}
```

而dispatchAttachedToWindow方法的执行，是依赖于View.addView方法的：

```
public void addView(View child) {
        addView(child, -1);
    }

public void addView(View child, int index) {
        ........
        addView(child, index, params);
    }

public void addView(View child, int index, LayoutParams params) {
        ........
        addViewInner(child, index, params, false);
    }

private void addViewInner(View child, int index, LayoutParams params,
            boolean preventRequestLayout) {
        ........
        AttachInfo ai = mAttachInfo;
        if (ai != null && (mGroupFlags & FLAG_PREVENT_DISPATCH_ATTACHED_TO_WINDOW) == 0) {
            boolean lastKeepOn = ai.mKeepScreenOn;
            ai.mKeepScreenOn = false;

            //在此调用了自定义View的方法
            child.dispatchAttachedToWindow(mAttachInfo, (mViewFlags&VISIBILITY_MASK));
            if (ai.mKeepScreenOn) {
                needGlobalAttributesUpdate(true);
            }
            ai.mKeepScreenOn = lastKeepOn;
        }
        ........
    }
```

en......

看到这里是不是都明白了。。。