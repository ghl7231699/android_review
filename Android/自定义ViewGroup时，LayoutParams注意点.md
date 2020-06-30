### 20191122
#### 每日一问：自定义 ViewGroup 的时候，关于 LayoutParams 有哪些注意事项？

LayoutParams的主要左右就是用来协助ViewGroup进行布局的。

在自定义ViewGroup时，如果需要我们自定义一个LayoutParams的话，需要创建一个extends ViewGroup.LayoutParams的静态内部类（通用的做法）。

##### ViewGroup需要重写的方法
1. generateDefaultLayoutParams()
2. generateLayoutParams(ViewGroup.LayoutParams p) 
3. checkLayoutParams(ViewGroup.LayoutParams p)

第一个方法，顾名思义，返回默认的LayoutParams。

```
/**
     * Returns a set of default layout parameters. These parameters are requested
     * when the View passed to {@link #addView(View)} has no layout parameters
     * already set. If null is returned, an exception is thrown from addView.
     *
     * @return a set of default layout parameters or null
     */
    protected LayoutParams generateDefaultLayoutParams() {
        return new LayoutParams(LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT);
    }
```

看源码注释，这个方法会在添加子View是调用，如果该子View的LayoutParams为空，或者没有直接调用 有LayoutParams参数的addView方法的话。

第二个generateLayoutParams(ViewGroup.LayoutParams p) 方法。在ViewGroup添加子View的过程中，往往需要和第三个checkLayoutParams配合使用。

这个checkLayoutParams就是检查传进来的LayoutParams是否合法的，当然了，里面的逻辑是怎样的，就要看你怎么重写了，默认的实现是做个非空检查。如果检查到传进来的Params不合法的话，接着就会调用这个generateLayoutParams方法。

第三个方法， 虽然说你不重写，在一般情况下也不会报错，但还是建议重写，并且在里面判断这个传进来的Params，是不是你自己定义的Params。

我们平时在xml里写布局时，通常会用到一些ViewGroup提供的属性， 比如FrameLayout的layout_gravity属性、LinearLayout的layout_weight等等。 可以发现，不同的ViewGroup它们所支持的属性也不同，这个其实也就是自定义的LayoutParams属性了，

##### 如何自定义LayoutParams属性

首先，在自定义的ViewGroup中重写generateLayoutParams(AttributeSet attrs)方法，等下可以从attrs中拿到各个子View所设置的属性。

然后，类似于自定义View，在attrs.xml中去定义一些用到的属性，这个styleable的命名格式一般为: "ViewGroup名_Layout"

最后，我们就可以回到LayoutParams中，找到有(Context c, AttributeSet attrs)这个参数的构造方法，在这里面就可以获取到子View中设置的属性了。
