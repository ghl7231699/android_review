### 20191121
#### AppCompatTextView和TextView
1. compat库是如何将TextView替换为AppCompatTextVew的？
2. 为什么要进行替换
3. 根据替换的相关原理，我们可以做哪些事情

先回答第一个问题。

TextView在运行时被替换成AppCompatView的前提是：该Activity必须继承自AppCompatActivity。

##### 如何替换的

我们给Activity设置布局一般会使用setContentView方法，打开AppCompatActivity，查看源码：

```
@Override
    public void setContentView(@LayoutRes int layoutResID) {
        getDelegate().setContentView(layoutResID);
    }
```

发现调用的是getDelegate()的setContentView方法，getDelegate()返回的是个AppCompatDelegate对象。AppCompatDelegate是个抽象类，所以setContentView 的最终实现还需要查看AppCompatDelegate的实现类。

在appcompat库中AppCompatDelegate的实现类有很多AppCompatDelegateImplV9、AppCompatDelegateImplV11、AppCompatDelegateImplV14、AppCompatDelegateImplV23、AppCompatDelegateImplN（基于SDK 27的源码）。但都是extends AppCompatDelegateImplV9。

查看AppCompatDelegateImplV9的setContentView方法。

```
@Override
    public void setContentView(View v) {
        ensureSubDecor();
        ViewGroup contentParent = (ViewGroup) mSubDecor.findViewById(android.R.id.content);
        contentParent.removeAllViews();
        contentParent.addView(v);
        mOriginalWindowCallback.onContentChanged();
    }
```
以上代码可以看到就是把传进去的布局Id，给Inflate出来并且把它添加到contentParent上。

回到AppCompatActivity，看下onCreate方法。

```
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        final AppCompatDelegate delegate = getDelegate();
        delegate.installViewFactory();
        delegate.onCreate(savedInstanceState);
.......
```

在AppCompatDelegateImplV9中查看installViewFactory方法。

```
 @Override
    public void installViewFactory() {
        LayoutInflater layoutInflater = LayoutInflater.from(mContext);
        if (layoutInflater.getFactory() == null) {
            LayoutInflaterCompat.setFactory2(layoutInflater, this);
        } else {
            if (!(layoutInflater.getFactory2() instanceof AppCompatDelegateImplV9)) {
                Log.i(TAG, "The Activity's LayoutInflater already has a Factory installed"
                        + " so we can not install AppCompat's");
            }
        }
    }
```

通过以上代码可以看到，在这个方法中会给LayoutInflater设置一个Factory2，并且set的是this，说明AppCompatDelegateImplV9实现了interface Factory2的方法。

我们知道，当LayoutInflater在inflate布局的时候，会优先调用Factory2的onCreateView方法。

现在来看AppCompatDelegateImplV9中onCreateView方法的实现：

```
    /**
     * From {@link LayoutInflater.Factory2}.
     */
    @Override
    public final View onCreateView(View parent, String name, Context context, AttributeSet attrs) {
        // First let the Activity's Factory try and inflate the view
        final View view = callActivityOnCreateView(parent, name, context, attrs);
        if (view != null) {
            return view;
        }

        // If the Factory didn't handle it, let our createView() method try
        return createView(parent, name, context, attrs);
    }
```

由以上看出调用的是createView方法：

```
 @Override
    public View createView(View parent, final String name, @NonNull Context context,
            @NonNull AttributeSet attrs) {
        if (mAppCompatViewInflater == null) {
            mAppCompatViewInflater = new AppCompatViewInflater();
        }

        boolean inheritContext = false;
        if (IS_PRE_LOLLIPOP) {
            inheritContext = (attrs instanceof XmlPullParser)
                    // If we have a XmlPullParser, we can detect where we are in the layout
                    ? ((XmlPullParser) attrs).getDepth() > 1
                    // Otherwise we have to use the old heuristic
                    : shouldInheritContext((ViewParent) parent);
        }

        return mAppCompatViewInflater.createView(parent, name, context, attrs, inheritContext,
                IS_PRE_LOLLIPOP, /* Only read android:theme pre-L (L+ handles this anyway) */
                true, /* Read read app:theme as a fallback at all times for legacy reasons */
                VectorEnabledTintResources.shouldBeUsed() /* Only tint wrap the context if enabled */
        );
    }
```

通过查看createView方法，发现最终调用的还是AppCompatViewInflater的createView。

```
 public final View createView(View parent, final String name, @NonNull Context context,
            @NonNull AttributeSet attrs, boolean inheritContext,
            boolean readAndroidTheme, boolean readAppTheme, boolean wrapContext) {
        final Context originalContext = context;

        // We can emulate Lollipop's android:theme attribute propagating down the view hierarchy
        // by using the parent's context
        if (inheritContext && parent != null) {
            context = parent.getContext();
        }
        if (readAndroidTheme || readAppTheme) {
            // We then apply the theme on the context, if specified
            context = themifyContext(context, attrs, readAndroidTheme, readAppTheme);
        }
        if (wrapContext) {
            context = TintContextWrapper.wrap(context);
        }

        View view = null;

        // We need to 'inject' our tint aware Views in place of the standard framework versions
        switch (name) {
            case "TextView":
                view = new AppCompatTextView(context, attrs);
                break;
            case "ImageView":
                view = new AppCompatImageView(context, attrs);
                break;
            case "Button":
                view = new AppCompatButton(context, attrs);
                break;
            case "EditText":
                view = new AppCompatEditText(context, attrs);
                break;
            case "Spinner":
                view = new AppCompatSpinner(context, attrs);
                break;
            case "ImageButton":
                view = new AppCompatImageButton(context, attrs);
                break;
            case "CheckBox":
                view = new AppCompatCheckBox(context, attrs);
                break;
            case "RadioButton":
                view = new AppCompatRadioButton(context, attrs);
                break;
            case "CheckedTextView":
                view = new AppCompatCheckedTextView(context, attrs);
                break;
            case "AutoCompleteTextView":
                view = new AppCompatAutoCompleteTextView(context, attrs);
                break;
            case "MultiAutoCompleteTextView":
                view = new AppCompatMultiAutoCompleteTextView(context, attrs);
                break;
            case "RatingBar":
                view = new AppCompatRatingBar(context, attrs);
                break;
            case "SeekBar":
                view = new AppCompatSeekBar(context, attrs);
                break;
        }

        if (view == null && originalContext != context) {
            // If the original context does not equal our themed context, then we need to manually
            // inflate it using the name so that android:theme takes effect.
            view = createViewFromTag(context, name, attrs);
        }

        if (view != null) {
            // If we have created a view, check its android:onClick
            checkOnClickListener(view, attrs);
        }

        return view;
    }
```

Bingo! 终于找到替换的位置了。

##### 为什么要替换

通过查看AppCompatXXX 控件的源码，我们发现都实现了TintableBackgroundView接口。源码如下：

```
/**
 * Interface which allows a {@link android.view.View} to receive background tinting calls from
 * {@link ViewCompat} when running on API v20 devices or lower.
 */
public interface TintableBackgroundView {

    /**
     * Applies a tint to the background drawable. Does not modify the current tint
     * mode, which is {@link PorterDuff.Mode#SRC_IN} by default.
     * <p>
     * Subsequent calls to {@code View.setBackground(Drawable)} will automatically
     * mutate the drawable and apply the specified tint and tint mode.
     *
     * @param tint the tint to apply, may be {@code null} to clear tint
     *
     * @see #getSupportBackgroundTintList()
     */
    void setSupportBackgroundTintList(@Nullable ColorStateList tint);

    /**
     * Return the tint applied to the background drawable, if specified.
     *
     * @return the tint applied to the background drawable
     */
    @Nullable
    ColorStateList getSupportBackgroundTintList();

    /**
     * Specifies the blending mode used to apply the tint specified by
     * {@link #setSupportBackgroundTintList(ColorStateList)}} to the background
     * drawable. The default mode is {@link PorterDuff.Mode#SRC_IN}.
     *
     * @param tintMode the blending mode used to apply the tint, may be
     *                 {@code null} to clear tint
     * @see #getSupportBackgroundTintMode()
     */
    void setSupportBackgroundTintMode(@Nullable PorterDuff.Mode tintMode);

    /**
     * Return the blending mode used to apply the tint to the background
     * drawable, if specified.
     *
     * @return the blending mode used to apply the tint to the background
     *         drawable
     */
    @Nullable
    PorterDuff.Mode getSupportBackgroundTintMode();
}
```

替换成AppCompat系列的，就是为了能够让旧版本(5.0以下)能够兼容一个叫BackgroundTint的东西:背景色。

这个东西有什么作用呢？

就是为控件设置背景！

##### 可以做什么

广为人知的就是主题替换了。

换肤、换字体、全局替换view。

#### 传送门

其实整个替换从以上所示的源码中可以看到，能够被替换的关键是Factory2存在，那么我觉得，其实问题问的是Factory2可以用来做什么吧？

[Factory传送门](https://blog.csdn.net/lmj623565791/article/details/51503977)



