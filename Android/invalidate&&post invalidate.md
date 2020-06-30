### 20191113
#### 请谈一谈invalidate()和postInvalidate()方法的区别和应用场景
  1. invalidate()方法，主要用于UI线程的刷新。一切以源码服人：

```
	/**
     * Invalidate the whole view. If the view is visible,
     * {@link #onDraw(android.graphics.Canvas)} will be called at some point in
     * the future.
     * <p>
     * This must be called from a UI thread. To call from a non-UI thread, call
     * {@link #postInvalidate()}.
     */
    public void invalidate() {
        invalidate(true);
    }
    
     /**
     * This is where the invalidate() work actually happens. A full invalidate()
     * causes the drawing cache to be invalidated, but this function can be
     * called with invalidateCache set to false to skip that invalidation step
     * for cases that do not need it (for example, a component that remains at
     * the same dimensions with the same content).
     *
     * @param invalidateCache Whether the drawing cache for this view should be
     *            invalidated as well. This is usually true for a full
     *            invalidate, but may be set to false if the View's contents or
     *            dimensions have not changed.
     * @hide
     */
    public void invalidate(boolean invalidateCache) {
        invalidateInternal(0, 0, mRight - mLeft, mBottom - mTop, invalidateCache, true);
    }
    
     void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache,
            boolean fullInvalidate) {
        if (mGhostView != null) {
            mGhostView.invalidate(true);
            return;
        }

        if (skipInvalidate()) {
            return;
        }

        if ((mPrivateFlags & (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)) == (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)
                || (invalidateCache && (mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == PFLAG_DRAWING_CACHE_VALID)
                || (mPrivateFlags & PFLAG_INVALIDATED) != PFLAG_INVALIDATED
                || (fullInvalidate && isOpaque() != mLastIsOpaque)) {
            if (fullInvalidate) {
                mLastIsOpaque = isOpaque();
                mPrivateFlags &= ~PFLAG_DRAWN;
            }

            mPrivateFlags |= PFLAG_DIRTY;

            if (invalidateCache) {
                mPrivateFlags |= PFLAG_INVALIDATED;
                mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
            }

            // Propagate the damage rectangle to the parent view.
            final AttachInfo ai = mAttachInfo;
            final ViewParent p = mParent;
            if (p != null && ai != null && l < r && t < b) {
                final Rect damage = ai.mTmpInvalRect;
                damage.set(l, t, r, b);
                p.invalidateChild(this, damage);
            }

            // Damage the entire projection receiver, if necessary.
            if (mBackground != null && mBackground.isProjected()) {
                final View receiver = getProjectionReceiver();
                if (receiver != null) {
                    receiver.damageInParent();
                }
            }
        }
    }

```

  2. postInvalidate()也是用来重新绘制UI,它可以在UI线程调用，也可以在子线程中调用，postInvalidate()方法内部通过Handler发送了一个消息将线程切回到UI线程通知重新绘制，并不是说postInvalidate()可以在子线程更新UI,本质上还是在UI线程发生重绘，只不过我们使用postInvalidate()它内部会帮我们切换线程。

源码如下：

```
/**
     * <p>Cause an invalidate to happen on a subsequent cycle through the event loop.
     * Use this to invalidate the View from a non-UI thread.</p>
     *
     * <p>This method can be invoked from outside of the UI thread
     * only when this View is attached to a window.</p>
     *
     * @see #invalidate()
     * @see #postInvalidateDelayed(long)
     */
    public void postInvalidate() {
        postInvalidateDelayed(0);
    }
    
    /**
     * <p>Cause an invalidate to happen on a subsequent cycle through the event
     * loop. Waits for the specified amount of time.</p>
     *
     * <p>This method can be invoked from outside of the UI thread
     * only when this View is attached to a window.</p>
     *
     * @param delayMilliseconds the duration in milliseconds to delay the
     *         invalidation by
     *
     * @see #invalidate()
     * @see #postInvalidate()
     */
    public void postInvalidateDelayed(long delayMilliseconds) {
        // We try only with the AttachInfo because there's no point in invalidating
        // if we are not attached to our window
        final AttachInfo attachInfo = mAttachInfo;
        if (attachInfo != null) {
            attachInfo.mViewRootImpl.dispatchInvalidateDelayed(this, delayMilliseconds);
        }
    }
    
     public void dispatchInvalidateDelayed(View view, long delayMilliseconds) {
        Message msg = mHandler.obtainMessage(MSG_INVALIDATE, view);
        mHandler.sendMessageDelayed(msg, delayMilliseconds);
    }
    
```

通过以上源码可以发现，postInvalidate()通过ViewRootHandler消息切换到主线程中，最终还是通过invaidate()方法进行UI的绘制更新。不管是invalidate，还是postInvalidate只有当view可见的时候才会进行view的更新绘制，且只会调用onDraw方法，不会调用onMeasure和onLayout方法。

#### 使用场景

一般情况下，如果View确定自身不再适合当前区域，比如说LayoutParams发生了变化，需要父布局对其重新进行测量、布局、绘制这三个流程，往往使用requestLayout,而如果只涉及到View的绘制，invalidate的效率要远远高于requestLayout。