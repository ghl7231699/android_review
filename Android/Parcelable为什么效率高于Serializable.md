### Parcelable为什么效率高于Serializable
1.  为什么效率比Serializable高？
2.  为了效率损失了什么？
3.  一个对象可以序列化的关键是什么？

以下统一用P来替代Parcelable，S来替代Serializable。

#### 先回答第一个，为什么效率高。

可以从设计目的和实现原理两个方面来分析。

设计目的

S是Java API，是一个通用的序列化机制，通过将文件保存到本地文件、网络流等实现数据的传递，这种传递不仅可以在单个程序中进行，也可以在两个不同的程序中进行；

P是Android SDK API，为了在同个程序的不同组件之间或不同程序（AIDL）之间高效传递数据，是通过IBInder通信的消息的载体。从设计上来看，P是为了Android高效传输数据而生的。

实现原理

S使用到了反射机制，序列化过程较慢，且在序列化过程中会创建许多临时对象，会引发频繁的gc.

P直接在内存中读写，自己实现了封送和解封（marshalled & unmarshalled）操作，不需要进行反射操作，数据存放在Native内存中，所以P具有效率高，内存开销小的优点。

#### 为了效率损失了什么？

S是通用的序列化机制，将数据存储在磁盘，可以做到持久化保存，文件的生命周期不受程序影响。

P的序列化操作完全由底层实现，不同的Android版本实现方式可能不相同，不能进行持久化保存。

#### 对象可以序列化的关键

序列化是将一个对象从存储状态转化为传输态的过程，把对象转换成字节序列，该字节序列包括该对象的数据，有关对象的类型的信息和存储在对象中数据的类型。

在序列化时，对象的各属性都必须是可序列化的，声明为static和Transient类型的成员数据不能被序列化。

并非所有的对象都可以序列化，有很多原因，比如：

1.安全方面的原因，比如一个对象拥有private，public等field,对于一个要传输的对象，比如写到文件，或者进行rmi传输等，在序列化进行传输的过程中，这个对象的private等域是不受保护的。

2.资源分配方面的原因，比如socket、thread类，如果可以序列化，进行传输或者保存，也无法对他们进行重新的资源分配，而且也没有必要这样实现。


### 高光时刻

通过查看两种方式的源码，我们会发现Serializable的Object(input,output)Stream，都是直接在java层实现，而Parcelable的主要逻辑都是通过native方法实现的。

Serializable原理：

writeObject方法流程大致如下：
1. 借助ObjectStreamClass记录目标对象的类型、类名等信息，这个类里面还有个ObjectStreamField[]数组，用来记录目标对象的内部变量；
2. 在ObjectOutputStream的defaultWriteFields方法中，会先通过ObjectStreamClass的getPrimFieldValues方法，把基本数据类型的值都复制到一个叫primVals的数组上；
3. 然后通过getPrimFieldValues方法来获取所有成员变量的值，***出乎意料的是：这两个获取值的方法，里面不是用的反射操作（Field.get）,而是通过操作UnSafe类来完成的***；
4. 遍历余下的不是基本数据类型的成员变量，然后递归调用writeObject方法（也就是一层一层的剥开目标对象，直到找到基本数据类型为止）；

readObject的流程类似：先通过readClassDescriptor方法(里面会把InputStream里面的数据读出来)获取到ObjectStreamClass 的实例，再根据这个实例来把基本数据类型和引用数据类型分别赋值到用反射创建的目标对象实例中。

**所以从JDK 1.8的源码来看，Serializable序列化的时候是没有反射操作的，反序列化的时候才用到了反射。**

至于其他jdk版本，请大家自行查看相关源码。。

Parcelable原理：

各种writexxx方法，在native层都是会调用[Parcel.cpp的write方法](http://androidxref.com/9.0.0_r3/xref/system/libhwbinder/Parcel.cpp#write)，它是直接复制内存（memcpy），地址由一个叫mData的uint8_t来保存。

[read方法同理](http://androidxref.com/9.0.0_r3/xref/system/libhwbinder/Parcel.cpp#read)，它也是通过memcpy函数来把mData上的某部分数据复制出来。