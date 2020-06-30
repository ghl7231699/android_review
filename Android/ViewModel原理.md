# 20191218
### ViewModel的出现是为了解决什么问题？并简要说说它的内部原理

看下viewModel的优点就知道了：

1.对于activity/fragment的销毁重建，它们内部的数据也会销毁，通常可以用onSaveInstanceState()防法保存，通过onCreate的bundle中重新获取，但是大量的数据不合适，而vm会再页面销毁时自动保存并在页面加载时恢复。

2.对于异步获取数据，大多时候会在页面destroyed时回收资源，随着数据和资源的复杂，会造成页面中的回收操作越来越多，页面处理ui的同时还要处理资源和数据的管理。而引入vm后可以把资源和数据的处理统一放在vm里，页面回收时系统也会回收vm。加上databinding的支持后，会大幅度分担ui层的负担。

内部原理：

vm内部很简单，只有一个onClean方法。

vm的创建一般是这样
```
ViewModelProviders.of(getActivity()).get(UserModel.class);
```

1.ViewModelProviders.of(getActivity())
在of方法中通过传入的activity获取构造一个HolderFragment，HolderFragment内有个ViewModelStore，而ViewModelStore内部的一个hashMap保存着系统构造的vm对象，HolderFragment可以感知到传入页面的生命周期（跟glide的做法差不多），HolderFragment构造方法中设置了setRetainInstance(true)，所以页面销毁后vm可以正常保存。

2.get(UserModel.class);

获取ViewModelStore.hashMap中的vm，第一次为空会走创建逻辑，如果我们没有提供vm创建的Factory，使用我们传入的activity获取application创建AndroidViewModelFactory，内部使用反射创建我们需要的vm对象。