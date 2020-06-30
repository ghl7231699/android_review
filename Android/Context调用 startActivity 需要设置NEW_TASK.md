### 20191119
#### 哪些 Context调用 startActivity 需要设置NEW_TASK，why？

##### 哪些context需要设置

BroadcastReceiver.onReceive(context)，Service，Application，ContextProvicder中的Context mBase实现类都是ContextImpl，ContextImpl类中的startActivity方法对intent.flag进行了检查，在这些组件中获得的context启动Activity需要制定新的任务栈，因为这些组件本身无任务栈。

接下来以application context为例来说明。

大多数人都知道我们使用非**activity的StartActivity（）**的时候，都需要指定Intent.FLAG_ACTIVITY_NEW_TASK，如果没有指定，直接进行操作则会直接抛出异常。



上面我们使用Application Context做**activity的StartActivity（）**操作，不出意外的引发了crash，正确的代码如下：

```
val intent = Intent(this,ChatMessageActivity::class.java)

intent.flags = Intent.FLAG_ACTIVITY_NEW_TASK

applicationContext.startActivity(intent)

```

上述的代码，有明显的问题，我们使用 applicationContext 来做 startActivity() 操作，却没有指定任何的 FLAG，但是，在 8.0 的手机上，你一定会惊讶的发现，我们并没有等到意料内的崩溃日志，而且跳转也是非常正常，这不由得和我们印象中必须加 FLAG 的结论大相径庭。然后再拿一个 9.0 的手机来尝试，马上就出现了上面的崩溃。

我们必须看看源码。我们先基于 SDK 26，直接打开 Context 的实现类 ContextImpl，直接通过关键字 context requires the FLAG_ACTIVITY_NEW_TASK flag 搜索定位到下面的方法。

```
@Override
public void startActivity(Intent intent, Bundle options) {
    warnIfCallingFromSystemProcess();

    // Calling start activity from outside an activity without FLAG_ACTIVITY_NEW_TASK is
    // generally not allowed, except if the caller specifies the task id the activity should
    // be launched in.
    if ((intent.getFlags()&Intent.FLAG_ACTIVITY_NEW_TASK) == 0
            && options != null && ActivityOptions.fromBundle(options).getLaunchTaskId() == -1) {
        throw new AndroidRuntimeException(
                "Calling startActivity() from outside of an Activity "
                + " context requires the FLAG_ACTIVITY_NEW_TASK flag."
                + " Is this really what you want?");
    }
    mMainThread.getInstrumentation().execStartActivity(
            getOuterContext(), mMainThread.getApplicationThread(), null,
            (Activity) null, intent, -1, options);
}
```
然后再打开SDK 28的源码：

```
@Override
public void startActivity(Intent intent, Bundle options) {
    warnIfCallingFromSystemProcess();

    // Calling start activity from outside an activity without FLAG_ACTIVITY_NEW_TASK is
    // generally not allowed, except if the caller specifies the task id the activity should
    // be launched in. A bug was existed between N and O-MR1 which allowed this to work. We
    // maintain this for backwards compatibility.
    final int targetSdkVersion = getApplicationInfo().targetSdkVersion;

    if ((intent.getFlags() & Intent.FLAG_ACTIVITY_NEW_TASK) == 0
            && (targetSdkVersion < Build.VERSION_CODES.N
                    || targetSdkVersion >= Build.VERSION_CODES.P)
            && (options == null
                    || ActivityOptions.fromBundle(options).getLaunchTaskId() == -1)) {
        throw new AndroidRuntimeException(
                "Calling startActivity() from outside of an Activity "
                        + " context requires the FLAG_ACTIVITY_NEW_TASK flag."
                        + " Is this really what you want?");
    }
    mMainThread.getInstrumentation().execStartActivity(
            getOuterContext(), mMainThread.getApplicationThread(), null,
            (Activity) null, intent, -1, options);
}

```

注释已经写的很清楚了，我们使用 Context.startActivity() 的时候是一定要加上 FLAG_ACTIVITY_NEW_TASK 的，但是在 Android N 到 O-MR1，即 24~27 之间却出现了 bug，即使没有加也会正确跳转。

对比源码发现，在我们非 Activity 调用 startActivity() 的时候，我们这个 options通常是 null 的，所以在 24~27 之间的时候，误把判断条件 options == null 写成了options != null 导致进不去 if，从而不会抛出异常。

##### 为什么没有NEW_TASK标志要拋异常呢？

可能Android它认为：

如果该Activity是由一个已启动的Activity发起的，那么把它放在这个已启动的Activity的任务栈内，这是合情合理的。

就好像生BB一样，如果是自己亲生的BB，当然是在自己家住了，但如果这个BB是从石头里爆出来的，那应该归谁家养呢？没人收养的话，就只有自己另起门户咯。

所以在这种情况下，就必须要你主动去承认这个Activity是在新的任务栈中，而不是那种妥协的方法：寄生在当前任务栈上面咯。

### 小Tips

至于非Activity启动但没NEW_TASK的处理，大家可以查看ActivityStarter的startActivityUnchecked() 源码。
