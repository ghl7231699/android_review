## 20191209
### 详细的描述下自定义View测量时MesureSpec.UNSPECIFIED

MeasureSpec.UNSPECIFIED，此种模式表示无限制，子元素告诉父容器它希望它的宽高想要多大就要多大，你不要限制我。一般的自定义view中，我们很少需要处理这种模式。

那么，这个模式什么时候会在onMeasure()里遇到呢？

其实是取决于它的父容器。就拿最常用的 RecyclerView 做例子，在  Item 进行 measure() 时，如果列表可滚动，并且  Item 的宽或高设置了 wrap_content 的话，那么接下来，itemView 的 onMeasure( )方法的测量模式就会变成  MeasureSpec.UNSPECIFIED。点开RecyclerView的源码，在measureChildWithMargins()方法中，会调用getChildMeasureSpec()方法。

```
public static int getChildMeasureSpec(int parentSize, int parentMode, int padding,
                int childDimension, boolean canScroll) {
            int size = Math.max(0, parentSize - padding);
            int resultSize = 0;
            int resultMode = 0;
            if (canScroll) {
                if (childDimension >= 0) {
                    resultSize = childDimension;
                    resultMode = MeasureSpec.EXACTLY;
                } else if (childDimension == LayoutParams.MATCH_PARENT) {
                    switch (parentMode) {
                        case MeasureSpec.AT_MOST:
                        case MeasureSpec.EXACTLY:
                            resultSize = size;
                            resultMode = parentMode;
                            break;
                        case MeasureSpec.UNSPECIFIED:
                            resultSize = 0;
                            resultMode = MeasureSpec.UNSPECIFIED;
                            break;
                    }
                } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                    resultSize = 0;
                    resultMode = MeasureSpec.UNSPECIFIED;
                }
            } else {
                if (childDimension >= 0) {
                    resultSize = childDimension;
                    resultMode = MeasureSpec.EXACTLY;
                } else if (childDimension == LayoutParams.MATCH_PARENT) {
                    resultSize = size;
                    resultMode = parentMode;
                } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                    resultSize = size;
                    if (parentMode == MeasureSpec.AT_MOST || parentMode == MeasureSpec.EXACTLY) {
                        resultMode = MeasureSpec.AT_MOST;
                    } else {
                        resultMode = MeasureSpec.UNSPECIFIED;
                    }

                }
            }
            //noinspection WrongConstant
            return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
        }
```
通过以上源码，我们可以看到，其中有一句注释：

```
MATCH_PARENT can't be applied since we can scroll in this dimension, wrap instead using UNSPECIFIED.
```
翻译过来就是 因为我们可以在这个维度中滚动，所以不能应用MATCH_PARENT，而是使用UNSPECIFIED模式.

**即如果child设置为WRAP_CONTENT,那么他的mode就会被设置成MeasureSpec.UNSPECIFIED.**

大家可能会有疑问：我明明设置的是WRAP__CONTENT，在onMeasure中应该收到AT_MOST才对啊，为什么要强转转换成UNSPECIFIED呢？ 

这是因为考虑到 Item 的尺寸有可能超出这个可滚动的 ViewGroup 的尺寸，而在 AT_MOST 模式下，你的尺寸不能超出你所在的 ViewGroup 的尺寸，最多只能等于，所以用 UNSPECIFIED会更合适，这个模式下你想要多大就多大。而且对于RecyclerView来说，他不需要限制view的大小，因为它可以滚动啊。

大家可以看下ScrollView，在它的实现中，不论childView的模式是哪个，最终都会强制转换成UNSPECIFIED的。

```
@Override
    protected void measureChild(View child, int parentWidthMeasureSpec,
            int parentHeightMeasureSpec) {
        ViewGroup.LayoutParams lp = child.getLayoutParams();

        int childWidthMeasureSpec;
        int childHeightMeasureSpec;

        childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec, mPaddingLeft
                + mPaddingRight, lp.width);
        final int verticalPadding = mPaddingTop + mPaddingBottom;
        childHeightMeasureSpec = MeasureSpec.makeSafeMeasureSpec(
                Math.max(0, MeasureSpec.getSize(parentHeightMeasureSpec) - verticalPadding),
                MeasureSpec.UNSPECIFIED);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```

那么如果我们在自定义View的时候，遇到UNSPECIFIED这种情况应该怎么处理呢？

这个就比较自由了，你可以随意设置，但还是要根据实际需要来做处理。

比如 ImageView，它的做法就是：有设置图片内容(drawable)的话，会直接使用这个 drawable 的尺寸，但不会超过指定的 MaxWidth 或  MaxHeight， 没有内容的话就是 0。而 TextView 处理  UNSPECIFIED 的方式，和 AT_MOST 是一样的。

当然了，这些尺寸都不一定等于最后layout出来的尺寸，因为最后决定子View位置和大小的，是在onLayout方法中，在这里你完全可以无视这些尺寸，去layout成自己想要的样子。不过，一般不会这么做。


## Tips

### MeasureSpec

我们知道view和viewGoup之间的测量是通过MeasureSpec类来做桥梁的。

```
public static class MeasureSpec {
        private static final int MODE_SHIFT = 30;
        private static final int MODE_MASK  = 0x3 << MODE_SHIFT;

        /** @hide */
        @IntDef({UNSPECIFIED, EXACTLY, AT_MOST})
        @Retention(RetentionPolicy.SOURCE)
        public @interface MeasureSpecMode {}

        public static final int UNSPECIFIED = 0 << MODE_SHIFT;

        public static final int EXACTLY     = 1 << MODE_SHIFT;

        public static final int AT_MOST     = 2 << MODE_SHIFT;

        public static int makeMeasureSpec(@IntRange(from = 0, to = (1 << MeasureSpec.MODE_SHIFT) - 1) int size,
                                          @MeasureSpecMode int mode) {
            if (sUseBrokenMakeMeasureSpec) {
                return size + mode;
            } else {
                return (size & ~MODE_MASK) | (mode & MODE_MASK);
            }
        }

        public static int makeSafeMeasureSpec(int size, int mode) {
            if (sUseZeroUnspecifiedMeasureSpec && mode == UNSPECIFIED) {
                return 0;
            }
            return makeMeasureSpec(size, mode);
        }

        @MeasureSpecMode
        public static int getMode(int measureSpec) {
            //noinspection ResourceType
            return (measureSpec & MODE_MASK);
        }

        public static int getSize(int measureSpec) {
            return (measureSpec & ~MODE_MASK);
        }

        static int adjust(int measureSpec, int delta) {
            final int mode = getMode(measureSpec);
            int size = getSize(measureSpec);
            if (mode == UNSPECIFIED) {
                // No need to adjust size for UNSPECIFIED mode.
                return makeMeasureSpec(size, UNSPECIFIED);
            }
            size += delta;
            if (size < 0) {
                Log.e(VIEW_LOG_TAG, "MeasureSpec.adjust: new size would be negative! (" + size +
                        ") spec: " + toString(measureSpec) + " delta: " + delta);
                size = 0;
            }
            return makeMeasureSpec(size, mode);
        }
    }
```

代码不是很多，我们需要重点关注的是:

```
MeasureSpec.UNSPECIFIED
MeasureSpec.EXACTLY
MeasureSpec.AT_MOST

MeasureSpec.makeMeasureSpec()
MeasureSpec.getMode()
MeasureSpec.getSize()
```

这三个静态常量和静态方法。

MeasureSpec 代表测量规则，而它的手段则是用一个 int 数值来实现。我们知道一个 int 数值有 32 bit。MeasureSpec 将它的高 2 位用来代表测量模式 Mode，低 30 位用来代表数值大小 Size。

[https://user-gold-cdn.xitu.io/2017/5/21/6d134a3d16b05ad033e5341d6981445a?imageView2/0/w/1280/h/960/format/webp/ignore-error/1](https://user-gold-cdn.xitu.io/2017/5/21/6d134a3d16b05ad033e5341d6981445a?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

通过makeMeasureSpec()方法将Mode和Size组合成一个measureSpec数值。
而通过getMode()和getSize()可以逆向的将一个measureSpec值解析成它的Mode和Size。