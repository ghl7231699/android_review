# 20190102
### pacelable通过bundle用于activity fragment间参数传递，存取是同一个对象吗？

大家都知道Activity同Fragment进行参数传递的时，Android提供了setArguments(Bundle args)和getArguments()方法。setArguments方法需要传递一个Budle对象，那么存取的这个Budle是同一个对象吗？

我们看下这两个方法的源码：

```
public void setArguments(@Nullable Bundle args) {
        if (mFragmentManager != null && isStateSaved()) {
            throw new IllegalStateException("Fragment already added and state has been saved");
        }
        mArguments = args;
    }
    
final public Bundle getArguments() {
        return mArguments;
    }
```

通过上面的源码，我们可以知道setArguments方法中直接将传进来的Budle对象赋值给了mArguments。

emmmm  同一个。。。。。

举个栗子：

在fragment中添加一个方法：

```
  public static ItemFragment newInstance(String content) {
        ItemFragment fragment = new ItemFragment();
        Bundle args = new Bundle();
        args.putString(ARG_CONTENT, content);
        value = new Person();
        Log.d(TAG, "newInstance: value" + value.hashCode());
        args.putParcelable(key, value);
        fragment.setArguments(args);
        return fragment;
    }
    
     @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        if (getArguments() != null) {
            mContent = getArguments().getString(ARG_CONTENT);
            Person person = getArguments().getParcelable(key);
            person.setName("大帅");

            Log.d(TAG, "newInstance: value" + person.hashCode() + value.getName());
        }
    }
```

在Activity中调用：

```
ItemFragment.newInstance("RecycleView1");
```
日志打印结果：

```
2020-01-02 11:14:07.850 5920-5920/com.mmc.sample D/ghl: newInstance: value223845066
2020-01-02 11:14:07.872 5920-5920/com.mmc.sample D/ghl: newInstance: value223845066大帅
```

这里我们比较的是hashCode，我们发现在getArguments方法获取到对象之后，进行了赋值操作，这时候打印传参，发现也同时被赋值。

debug模式下，可以看到是同一个对象