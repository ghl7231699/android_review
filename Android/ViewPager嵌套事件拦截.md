# 20191225
### ViewPager嵌套事件拦截

ViewPager嵌套ViewPager的情况下，不做任何处理的情况下，正常滑动非边界区域，默认会滑动内层的ViewPager。一般而言，事件分发会走**外层ViewPager**的onInterceptTouchEvent,内外都是ViewPager,代码都一样，为何事件没有被**外层ViewPager**拦截掉，反而交给了内层ViewPager呢？

#### ViewPager事件处理

1. ViewPager事件处理内容不多，主要就是左右翻页滑动的事件拦截，滑动事件又只需要拦截横向滑动。
2. 事件拦截处理相关的几个方法：dispatchTouchEvent,onInterceptTouchEvent;在ViewPager中只重写了onInterceptTouchEvent和onTouchEvent。

#### onInterceptTouchEvent

该方法主要计算是否拦截滑动事件变量：mIsBeingDragged，满足一下两个条件才会进行拦截：

1. 当xDiff * 0.5f > yDiff，简单说就是X轴上滑动的距离要大于Y轴上的2倍才拦截
2. 并且canScroll(View v, boolean checkV, int dx, int x, int y)方法返回false，该方法是判断子View能不能横向滑动，如果子View能滑动ViewPager就不拦截滑动

```
  if (dx != 0 && !isGutterDrag(mLastMotionX, dx)
                        && canScroll(this, false, (int) dx, (int) x, (int) y)) {
                    // Nested view has scrollable area under this point. Let it be handled there.
                    mLastMotionX = x;
                    mLastMotionY = y;
                    mIsUnableToDrag = true;
                    return false;
                }
                if (xDiff > mTouchSlop && xDiff * 0.5f > yDiff) {
                    if (DEBUG) Log.v(TAG, "Starting drag!");
                    mIsBeingDragged = true;
                    requestParentDisallowInterceptTouchEvent(true);
                    setScrollState(SCROLL_STATE_DRAGGING);
                    mLastMotionX = dx > 0
                            ? mInitialMotionX + mTouchSlop : mInitialMotionX - mTouchSlop;
                    mLastMotionY = y;
                    setScrollingCacheEnabled(true);
                } else if (yDiff > mTouchSlop) {
                    // The finger has moved enough in the vertical
                    // direction to be counted as a drag...  abort
                    // any attempt to drag horizontally, to work correctly
                    // with children that have scrolling containers.
                    if (DEBUG) Log.v(TAG, "Starting unable to drag!");
                    mIsUnableToDrag = true;
                }
```

canScroll源码：

```
/**
     * Tests scrollability within child views of v given a delta of dx.
     *
     * @param v View to test for horizontal scrollability
     * @param checkV Whether the view v passed should itself be checked for scrollability (true),
     *               or just its children (false).
     * @param dx Delta scrolled in pixels
     * @param x X coordinate of the active touch point
     * @param y Y coordinate of the active touch point
     * @return true if child views of v can be scrolled by delta of dx.
     */
    protected boolean canScroll(View v, boolean checkV, int dx, int x, int y) {
        if (v instanceof ViewGroup) {
            final ViewGroup group = (ViewGroup) v;
            final int scrollX = v.getScrollX();
            final int scrollY = v.getScrollY();
            final int count = group.getChildCount();
            // Count backwards - let topmost views consume scroll distance first.
            for (int i = count - 1; i >= 0; i--) {
                // TODO: Add versioned support here for transformed views.
                // This will not work for transformed views in Honeycomb+
                final View child = group.getChildAt(i);
                if (x + scrollX >= child.getLeft() && x + scrollX < child.getRight()
                        && y + scrollY >= child.getTop() && y + scrollY < child.getBottom()
                        && canScroll(child, true, dx, x + scrollX - child.getLeft(),
                                y + scrollY - child.getTop())) {
                    return true;
                }
            }
        }

        return checkV && v.canScrollHorizontally(-dx);
    }
```
可以看出这个方法是一个递归方法，它会先检查传进来的View是不是ViewGroup，如果是的话，就会把【符合条件的子View】继续扔进canScroll方法中。注意这时候参数checkV传的是true，表示这次要连这个View本身也一起检测，而非只检测它的子View。

那么符合条件的子View是指哪些呢？

即当前触摸的坐标点，在子View的边界范围内。

可以看到它会先加上ScrollX或ScrollY，就是为了触摸坐标能对齐到滚动后的View。

接着看最后一行，如果checkV为true的话，会调用v（也就是传进来的那个子View）的canScrollHorizontally方法，并将 负的dx 传了进去。

dx就是本次手指水平滑动的距离。如果是向右滑动的话，那么这个值就会是正数，向左滑则负数。

来看下canScrollHorizaontally的源码：

```
 public boolean canScrollHorizontally(int direction) {
        final int offset = computeHorizontalScrollOffset();
        final int range = computeHorizontalScrollRange() - computeHorizontalScrollExtent();
        if (range == 0) return false;
        if (direction < 0) {
            return offset > 0;
        } else {
            return offset < range - 1;
        }
    }
```

先看下后面的判断逻辑，如果是向右滑动（direction<0）,则判断offset>0，看下offset是如何来的：

```
protected int computeHorizontalScrollOffset() {
        return mScrollX;
    }
   
```

它是直接返回了ScrollX的值，也就是说，如果手指是向右滑动的话，就看这个View之前有没有向左边滚动过，有的话，就表示可以向右滚动（返回true）。

如果手指是向左滑动，则判断offset是否小于**range-1**,range的值可以看到是用computeHorizontalScrollRange方法的返回值 减去 computeHorizontalScrollExtent的返回值，看看这两个方法：

```
 protected int computeHorizontalScrollExtent() {
        return getWidth();
    }
    
 protected int computeVerticalScrollRange() {
        return getHeight();
    }
```

emmm,返回的是该View的宽度，那么range的值就是0了。

向左滑动的时候，参数direction会是正数，不可能会小于 -1了，所以这时候会返回true

#### onTouchEvent

该方法主要是通过上面计算出的mIsBeingDragged变量，判断是否需要滑动操作，在不同MotionEvent中处理的内容：

1. ACTION_MOVE：如果mIsBeingDragged = fasle，这里会重新计算，这里的判断条件是xDiff > yDiff就能滑动了，然后调用performDrag方法完成具体滑动操作
2. ACTION_UP：调用setCurrentItemInternal滑动到最终的页面
3. ACTION_CANCEL：跟ACTION_UP差不多，调用scrollToItem完事
