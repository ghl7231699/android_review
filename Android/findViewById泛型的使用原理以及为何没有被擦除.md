# 20191223
### findViewById泛型的使用原理以及为何没有被擦除

findViewById源码分析

```
@SuppressWarnings("TypeParameterUnusedInFormals")
    @Override
    public <T extends View> T findViewById(@IdRes int id) {
        return getDelegate().findViewById(id);
    }
    
    /**
     * @return The {@link AppCompatDelegate} being used by this Activity.
     */
    @NonNull
    public AppCompatDelegate getDelegate() {
        if (mDelegate == null) {
            mDelegate = AppCompatDelegate.create(this, this);
        }
        return mDelegate;
    }

```
点开源码，我们发现无论是Activity的findViewById或者是View的，最后调用的都是view的findViewById方法。

```
@Nullable
    public final <T extends View> T findViewById(@IdRes int id) 	{
        if (id == NO_ID) {
            return null;
        }
        return findViewTraversal(id);
    }

```

我们发现最终调用了findViewTraversal方法。我们来看下：

```
/**
     * @param id the id of the view to be found
     * @return the view of the specified id, null if cannot be found
     * @hide
     */
    protected <T extends View> T findViewTraversal(@IdRes int id) {
        if (id == mID) {
            return (T) this;
        }
        return null;
    }
```
如果要找的id等于当前的mId，则直接返回当前的view，否则返回null。

ViewGroup中也对这个方法进行了重写：

```
 @Override
    protected <T extends View> T findViewTraversal(@IdRes int id) {
    	 // 如果要找的id等于当前View的id就返回当前的View
        if (id == mID) {
            return (T) this;
        }

        final View[] where = mChildren;
        final int len = mChildrenCount;
		 // 遍历自己的子view
        for (int i = 0; i < len; i++) {
            View v = where[i];

            if ((v.mPrivateFlags & PFLAG_IS_ROOT_NAMESPACE) == 0) {				
            // 调用ziView的findViewById()
                v = v.findViewById(id);

                if (v != null) {
                    return (T) v;
                }
            }
        }

        return null;
    }
```

之前版本的findViewById方法，返回的是个View对象，然后再强转成各个子类，后来加入了方法泛型，省去了强转的操作。