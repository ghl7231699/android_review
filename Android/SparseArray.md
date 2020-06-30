### 20191203
#### 为什么推荐用SparseArray代替HashMap？

在Android项目中，当我们新建一个Key为Integer类型的HashMap时，as会提示我们使用SparseArray替代HashMap来获得更好地表现。HashMap已经跟牛逼的了，为什么Android官方还推荐使用SparseArray呢？

##### SparseArray

翻译过来就是**稀疏数组**。所谓**稀疏数组**就是数组中大部分的内容值都未被使用（或都为零），在数组中，仅有少部分的空间使用，因此造成内存空间的浪费。为了节省空间，并且不影响数组中原有的内容值，我们可以采用一种压缩的方式来表示稀疏数组的内容。

网上举例子的两张图：



![]()