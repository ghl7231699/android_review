### 20191128
#### ANR的产生的原理是什么，AMS中涉及ANR的代码有哪些？

如果我在UI线程中执行一个非常耗时的操作（>5s），一定会弹出ANR弹框吗？

带着我们的问题，开始撸源码。

从源码中来，到源码中去。

首先我们打开ActivityManagerService的源码，在其中会找到：

```
static final int SHOW_NOT_RESPONDING_UI_MSG = 2;
```

见名知意，通过名字我们就知道这是个程序无响应的标识。接下来看看哪些地方都调用了这个字段：

```
final class UiHandler extends Handler {
.....
 @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
            case SHOW_ERROR_UI_MSG: {
                mAppErrors.handleShowAppErrorUi(msg);
                ensureBootCompleted();
            } break;
            case SHOW_NOT_RESPONDING_UI_MSG: {
                mAppErrors.handleShowAnrUi(msg);
                ensureBootCompleted();
            } break;
           }
          }
.....

}
```

我们发现只在UIHandler的hanleMessage方法中，出现了这个标识。由源码看到是AppErrors对象调用了handleShowAppErrorUi(msg)方法。我们看handleShowAppErrorUi(msg)的具体实现：

```
void handleShowAnrUi(Message msg) {
        Dialog d = null;
        synchronized (mService) {
            HashMap<String, Object> data = (HashMap<String, Object>) msg.obj;
            ProcessRecord proc = (ProcessRecord)data.get("app");
            if (proc != null && proc.anrDialog != null) {
                Slog.e(TAG, "App already has anr dialog: " + proc);
                MetricsLogger.action(mContext, MetricsProto.MetricsEvent.ACTION_APP_ANR,
                        AppNotRespondingDialog.ALREADY_SHOWING);
                return;
            }

            Intent intent = new Intent("android.intent.action.ANR");
            if (!mService.mProcessesReady) {
                intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY
                        | Intent.FLAG_RECEIVER_FOREGROUND);
            }
            mService.broadcastIntentLocked(null, null, intent,
                    null, null, 0, null, null, null, AppOpsManager.OP_NONE,
                    null, false, false, MY_PID, Process.SYSTEM_UID, 0 /* TODO: Verify */);

            boolean showBackground = Settings.Secure.getInt(mContext.getContentResolver(),
                    Settings.Secure.ANR_SHOW_BACKGROUND, 0) != 0;
            if (mService.canShowErrorDialogs() || showBackground) {
                d = new AppNotRespondingDialog(mService,
                        mContext, proc, (ActivityRecord)data.get("activity"),
                        msg.arg1 != 0);
                proc.anrDialog = d;
            } else {
                MetricsLogger.action(mContext, MetricsProto.MetricsEvent.ACTION_APP_ANR,
                        AppNotRespondingDialog.CANT_SHOW);
                // Just kill the app if there is no dialog to be shown.
                mService.killAppAtUsersRequest(proc, null);
            }
        }
        // If we've created a crash dialog, show it without the lock held
        if (d != null) {
            d.show();
        }
    }

```

通过上面的源码，我们知道，这里就是弹出ANR的位置了。

那么这个anr消息又是如何发出来的呢？

全局搜索**SHOW_NOT_RESPONDING_UI_MSG**，我们发现还有一个地方引用了该字段：AppErrors。查看调用的位置：

```
synchronized (mService) {
            mService.mBatteryStatsService.noteProcessAnr(app.processName, app.uid);

            if (isSilentANR) {
                app.kill("bg anr", true);
                return;
            }

            // Set the app's notResponding state, and look up the errorReportReceiver
            makeAppNotRespondingLocked(app,
                    activity != null ? activity.shortComponentName : null,
                    annotation != null ? "ANR " + annotation : "ANR",
                    info.toString());

            // Bring up the infamous App Not Responding dialog
            Message msg = Message.obtain();
            HashMap<String, Object> map = new HashMap<String, Object>();
            msg.what = ActivityManagerService.SHOW_NOT_RESPONDING_UI_MSG;
            msg.obj = map;
            msg.arg1 = aboveSystem ? 1 : 0;
            map.put("app", app);
            if (activity != null) {
                map.put("activity", activity);
            }

            mService.mUiHandler.sendMessage(msg);
        }
```

。。。熟悉的代码，熟悉的套路！

可以看到，在这个方法里会给AMS的UiHandler发送请求弹出ANR对话框，那到底在哪些地方调用这个方法的呢？

目前看到调用**appNotResponding**方法的位置有：

1. AMS中的appNotRespondingViaProvider方法。
2. AMS中的inputDispatchingTimedOut方法。
3. ActiveServices的serviceTimeout方法。
4. ActivityServices的serviceForegroundTimeout方法。

那么这四个方法都是啥意思呢？

1. 第一个appNotRespondingViaProvider，通过Provider发出的无响应，看里面的代码会看到"ContentProvider not responding"，这个ANR也就是由ContentProvider造成的了。
2. 第二个inputDispatchingTimedOut，看注释：处理输入调度超时，就是input事件分派的时候超时(处理事件时被阻塞)所发出的，input事件，我们知道分两种，一种是KeyEvent(按键)，另一种是MotionEvent(触摸)。
3. serviceTimeout，。。。。。这就是后台service服务超时的发出的啊 。
4. serviceForegroundTimeout。。这应该就是前台服务超时了。

再来回答我们刚才的问题：

#####如果我们在UI线程执行一个耗时操作（>5s），就一定会出现anr弹框吗？

来看AMS的inputDispatchingTimedOut方法。这个应该是我们关注的重点。。

```
  /**
     * Handle input dispatching timeouts.
     * Returns whether input dispatching should be aborted or not.
     */
    public boolean inputDispatchingTimedOut(final ProcessRecord proc,
            final ActivityRecord activity, final ActivityRecord parent,
            final boolean aboveSystem, String reason) {
        if (checkCallingPermission(android.Manifest.permission.FILTER_EVENTS)
                != PackageManager.PERMISSION_GRANTED) {
            throw new SecurityException("Requires permission "
                    + android.Manifest.permission.FILTER_EVENTS);
        }

        final String annotation;
        if (reason == null) {
            annotation = "Input dispatching timed out";
        } else {
            annotation = "Input dispatching timed out (" + reason + ")";
        }

        if (proc != null) {
            synchronized (this) {
                if (proc.debugging) {
                    return false;
                }

                if (proc.instr != null) {
                    Bundle info = new Bundle();
                    info.putString("shortMsg", "keyDispatchingTimedOut");
                    info.putString("longMsg", annotation);
                    finishInstrumentationLocked(proc, Activity.RESULT_CANCELED, info);
                    return true;
                }
            }
            mHandler.post(new Runnable() {
                @Override
                public void run() {
                    mAppErrors.appNotResponding(proc, activity, parent, aboveSystem, annotation);
                }
            });
        }

        return true;
    }
```

由源码可以看到，在倒数第二行开启子线程调用mAppErrors.appNotResponding(proc, activity, parent, aboveSystem, annotation);

同时，我们发现，在发送消息之前有过多个检查，会直接return：

1.proc.debugging==true的时候，会直接return false；这是啥意思呢？
```
 boolean debugging;          // was app launched for debugging?
```
看注释我们就知道，是否是调试模式。如果是调试模式的话，及时input事件超时了，也不会弹出anr。  
2. proc.instr != null的话，会执行finishInstrumentationLocked方法，你没看错，直接kill 重启。是不是很熟悉的场景。

至于这个proc.instr是什么，篇幅有限，大家可以自行去查看下。

#####总结:
ANR对话框，是在AMS收到SHOW_NOT_RESPONDING_UI_MSG消息之后弹出的。这个消息分别来自于：

ContentProvider超时；

input事件分派超时；

前台服务超时；

后台服务超时；

在input事件分派超时的时候，有两种情况不会弹框，分别是：

处于debug时；

来自子进程；(这个情况下会直接kill掉子进程)


### 什么情况下instr不为空呢？

它是当AMS的attachApplication方法被调用时，有可能被赋值。

####什么时候attachApplication会被调用呢？

其实就是ActivityThread的main方法执行的时候(启动)，它会调用一个attach方法，而attachApplication会在这个attach方法里面被调用。

####赋值条件是什么？
赋值条件是mActiveInstrumentation里面不为空。

####那么什么时候mActiveInstrumentation里面不为空？
看AMS代码70653行(检查mActiveInstrumentation是否为空那里)可以发现有一句注释："Check if this is a secondary process"，
那我们可以知道，这个判断是检测当前进程是否次进程(多进程环境下)，如果是次线程(不是主进程)，那么经过安全检查之后，就会把mActiveInstrumentation里面的实例，赋值给proc的instr。

### 以上源码都是基于android-27的