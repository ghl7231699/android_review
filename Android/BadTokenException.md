### 20191111
#### 每日一问：关于BadTokenException 异常，哪些场景下会出现这个异常？分别如何解决？
* 哪些场景下会出现这个异常？

源码分析

以android-27源码为例说明。

```
int res; /* = WindowManagerImpl.ADD_OKAY; */
.......
 res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,getHostVisibility(), 
 mDisplay.getDisplayId(),
 mAttachInfo.mContentInsets,
 mAttachInfo.mStableInsets,
 mAttachInfo.mOutsets, 
 mInputChannel);
 .......
 
 if (res < WindowManagerGlobal.ADD_OKAY) {
   mAttachInfo.mRootView = null;
   mAdded = false;
   mFallbackEventHandler.setView(null);
   unscheduleTraversals();
   setAccessibilityFocus(null, null);
   switch (res) {
                        case WindowManagerGlobal.ADD_BAD_APP_TOKEN:
                        case WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN:
                            throw new WindowManager.BadTokenException(
                                    "Unable to add window -- token " + attrs.token
                                    + " is not valid; is your activity running?");
                        case WindowManagerGlobal.ADD_NOT_APP_TOKEN:
                            throw new WindowManager.BadTokenException(
                                    "Unable to add window -- token " + attrs.token
                                    + " is not for an application");
                        case WindowManagerGlobal.ADD_APP_EXITING:
                            throw new WindowManager.BadTokenException(
                                    "Unable to add window -- app for token " + attrs.token
                                    + " is exiting");
                        case WindowManagerGlobal.ADD_DUPLICATE_ADD:
                            throw new WindowManager.BadTokenException(
                                    "Unable to add window -- window " + mWindow
                                    + " has already been added");
                        case WindowManagerGlobal.ADD_STARTING_NOT_NEEDED:
                            // Silently ignore -- we would have just removed it
                            // right away, anyway.
                            return;
                        case WindowManagerGlobal.ADD_MULTIPLE_SINGLETON:
                            throw new WindowManager.BadTokenException("Unable to add window "
                                    + mWindow + " -- another window of type "
                                    + mWindowAttributes.type + " already exists");
                        case WindowManagerGlobal.ADD_PERMISSION_DENIED:
                            throw new WindowManager.BadTokenException("Unable to add window "
                                    + mWindow + " -- permission denied for window type "
                                    + mWindowAttributes.type);
                        case WindowManagerGlobal.ADD_INVALID_DISPLAY:
                            throw new WindowManager.InvalidDisplayException("Unable to add window "
                                    + mWindow + " -- the specified display can not be found");
                        case WindowManagerGlobal.ADD_INVALID_TYPE:
                            throw new WindowManager.InvalidDisplayException("Unable to add window "
                                    + mWindow + " -- the specified window type "
                                    + mWindowAttributes.type + " is not valid");
                    }
                    throw new RuntimeException(
                            "Unable to add window -- unknown error code " + res);
                }

```
由源码可以看到，是通过IWindowSession的addToDisplay方法的返回值做判断的。这个方法最终调用的是WMS的addWindow()方法。

来看下addWindwo()方法中在什么情况下会返回以上7中flag。以ADD_BAD_APP_TOKEN为例：

```
// Use existing parent window token for child windows since they go in the same token
// as there parent window so we can apply the same policy on them.
 WindowToken token = displayContent.getWindowToken(hasParent ? parentWindow.mAttrs.token : attrs.token);
 // If this is a child window, we want to apply the same type checking rules as the
 // parent window type.
final int rootType = hasParent ? parentWindow.mAttrs.type : type;

boolean addToastWindowRequiresToken = false;

            if (token == null) {
                if (rootType >= FIRST_APPLICATION_WINDOW && rootType <= LAST_APPLICATION_WINDOW) {
                    Slog.w(TAG_WM, "Attempted to add application window with unknown token "
                          + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (rootType == TYPE_INPUT_METHOD) {
                    Slog.w(TAG_WM, "Attempted to add input method window with unknown token "
                          + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (rootType == TYPE_VOICE_INTERACTION) {
                    Slog.w(TAG_WM, "Attempted to add voice interaction window with unknown token "
                          + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (rootType == TYPE_WALLPAPER) {
                    Slog.w(TAG_WM, "Attempted to add wallpaper window with unknown token "
                          + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (rootType == TYPE_DREAM) {
                    Slog.w(TAG_WM, "Attempted to add Dream window with unknown token "
                          + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (rootType == TYPE_QS_DIALOG) {
                    Slog.w(TAG_WM, "Attempted to add QS dialog window with unknown token "
                          + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (rootType == TYPE_ACCESSIBILITY_OVERLAY) {
                    Slog.w(TAG_WM, "Attempted to add Accessibility overlay window with unknown token "
                            + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (type == TYPE_TOAST) {
                    // Apps targeting SDK above N MR1 cannot arbitrary add toast windows.
                    if (doesAddToastWindowRequireToken(attrs.packageName, callingUid,
                            parentWindow)) {
                        Slog.w(TAG_WM, "Attempted to add a toast window with unknown token "
                                + attrs.token + ".  Aborting.");
                        return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                    }
                }
                final IBinder binder = attrs.token != null ? attrs.token : client.asBinder();
                token = new WindowToken(this, binder, type, false, displayContent,
                        session.mCanAddInternalSystemWindow);
} else if (rootType >= FIRST_APPLICATION_WINDOW && rootType <= LAST_APPLICATION_WINDOW) {
                atoken = token.asAppWindowToken();
                if (atoken == null) {
                    Slog.w(TAG_WM, "Attempted to add window with non-application token "
                          + token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_NOT_APP_TOKEN;
                } else if (atoken.removed) {
                    Slog.w(TAG_WM, "Attempted to add window with exiting application token "
                          + token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_APP_EXITING;
                }
            } else if (rootType == TYPE_INPUT_METHOD) {
                if (token.windowType != TYPE_INPUT_METHOD) {
                    Slog.w(TAG_WM, "Attempted to add input method window with bad token "
                            + attrs.token + ".  Aborting.");
                      return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
            } else if (rootType == TYPE_VOICE_INTERACTION) {
                if (token.windowType != TYPE_VOICE_INTERACTION) {
                    Slog.w(TAG_WM, "Attempted to add voice interaction window with bad token "
                            + attrs.token + ".  Aborting.");
                      return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
            } else if (rootType == TYPE_WALLPAPER) {
                if (token.windowType != TYPE_WALLPAPER) {
                    Slog.w(TAG_WM, "Attempted to add wallpaper window with bad token "
                            + attrs.token + ".  Aborting.");
                      return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
            } else if (rootType == TYPE_DREAM) {
                if (token.windowType != TYPE_DREAM) {
                    Slog.w(TAG_WM, "Attempted to add Dream window with bad token "
                            + attrs.token + ".  Aborting.");
                      return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
            } else if (rootType == TYPE_ACCESSIBILITY_OVERLAY) {
                if (token.windowType != TYPE_ACCESSIBILITY_OVERLAY) {
                    Slog.w(TAG_WM, "Attempted to add Accessibility overlay window with bad token "
                            + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
            } else if (type == TYPE_TOAST) {
                // Apps targeting SDK above N MR1 cannot arbitrary add toast windows.
                addToastWindowRequiresToken = doesAddToastWindowRequireToken(attrs.packageName,
                        callingUid, parentWindow);
                if (addToastWindowRequiresToken && token.windowType != TYPE_TOAST) {
                    Slog.w(TAG_WM, "Attempted to add a toast window with bad token "
                            + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
            } else if (type == TYPE_QS_DIALOG) {
                if (token.windowType != TYPE_QS_DIALOG) {
                    Slog.w(TAG_WM, "Attempted to add QS dialog window with bad token "
                            + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
            } else if (token.asAppWindowToken() != null) {
                Slog.w(TAG_WM, "Non-null appWindowToken for system window of rootType=" + rootType);
                // It is not valid to use an app token with other system types; we will
                // instead make a new token for it (as if null had been passed in for the token).
                attrs.token = null;
                token = new WindowToken(this, client.asBinder(), type, false, displayContent,
                        session.mCanAddInternalSystemWindow);
            }

```

这里面的两个type(type&rootType)就是调用WindowManager的addView方法时，传进去的LayoutParams的type，默认是TYPE_APPLICATION。所以后面的else if就是检查窗口类型是否匹配了。

那这里的这个token什么时候为空呢？

从源码可以看到，token是通过DisplayContent的getWindowToken()方法获得的（这些WindowToken都保存在DisplayContent的HashMap<IBinder, WindowToken>中）

```
WindowToken getWindowToken(IBinder binder) {
        return mTokenMap.get(binder);
    }
```

它是直接调用Map的get方法，如果返回null的话，很可能是被移除了，搜一下"mTokenMap.remove"，会看到在removeWindowToken方法有调用：

```
   WindowToken removeWindowToken(IBinder binder) {
        final WindowToken token = mTokenMap.remove(binder);
        if (token != null && token.asAppWindowToken() == null) {
            token.setExiting();
        }
        return token;
    }

```

removeWindowToken何时被调用呢？
顺腾摸瓜，可以找到如下调用的位置

```
DisplayContent.removeAppToken() ->    
AppWindowContainerController.removeContainer() ->    
ActivityStack.removeWindowContainer() ->  
ActivityStack.removeActivityFromHistoryLocked() ->
ActivityStack.activityDestroyedLocked() ->
ActivityManagerService.activityDestroyed() ->
ActivityThread.handleDestroyActivity()

```
emmm，现在就很清晰了：
当Activity被Destroy的时候就会调用到removeWindowToken方法，把对应的WindowToken移除，
调用remove方法所传进去的key（IBinder），其实就是ActivityThread调用Activity的attach方法时传进来的，
也就是Activity.mToken了，也是这个Activity所对应的PhoneWindow里面的mAppToken，
这个Activity所对应的WindowManager，它里面也会持有Activity的PhoneWindow，
当调用WindowManager的addView方法时，还会把PhoneWindow的mAppToken赋值给WindowManager.LayoutParams.token！

回到上面那行代码

```
// attrs就是调用WindowManager的`addView`方法传进去的LayoutParams
WindowToken token = displayContent.getWindowToken(attrs.token);

```

此时的attrs.token就是Activity的mToken，当这个Activity被Destroy之后，再通过它的WindowManager添加View，最终就会抛出BadTokenException，这也就对应了开头对这个异常的描述：" is your activity running?"。

至于剩下的flag，需要大家自己动手去源码里查看了。。。

#### 常见场景

1. 列表在“加载更多”的时候，退出了该Activity，从后台拉取数据成功时，Activity已经被Destroy了。（对应Flag: ADD_BAD_APP_TOKEN）

2. 在Service中使用WindowManager添加悬浮窗 或者，弹出对话框时，使用的是默认的TYPE_APPLICATION类型，即没有指定View类型为SYSTEM_WINDOW。（对应Flag: ADD_NOT_APP_TOKEN）

3. 在Activity中的showDialog方法，使用了ApplicationContext

4. 7.1.1版本，在调用Toast的show方法之后，主线程卡住的时间 > 该Toast的显示时长。（对应Flag: ADD_BAD_APP_TOKEN）

5. 8.0之后弹出系统弹窗的时候，如果没有设置android.permission.SYSTEM_ALERT_WINDOW权限和WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY。（Flag：ADD_PERMISSION_DENIED）
