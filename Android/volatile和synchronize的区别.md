# volatile和synchronize的区别

### 原子性（atomicity）和可见性（visibility）

原子性意味着**一个时刻，只有一个线程能够执行一段代码**，这段代码通过一个monitor object保护。从而防止多个线程在更新状态时相互冲突。

可见性则更为微妙，它必须确保释放锁之前对共享数据作出的更改对于随后获得该锁的另一个线程是可见的。如果没有同步机制提供的这种可见性保证，线程看到的共享变量可能是修改前的值或不一致的值，这将引发许多严重问题。

### volatile

volatile 是一个类型修饰符。volatile 的作用是作为指令关键字，确保本条指令不会因编译器的优化而省略。

它所修饰的**变量不保留拷贝，直接访问内存中的**。

在Java内存模型中，有**main memory，每个线程也有自己的memory (例如寄存器)。为了性能，一个线程会在自己的memory中保持要访问的变量的副本。这样就会出现同一个变量在某个瞬间，在一个线程的memory中的值可能与另外一个线程memory中的值，或者main memory中的值不一致的情况。 一个变量声明为volatile，就意味着这个变量是随时会被其他线程修改的，因此不能将它cache在线程memory中**。

### 使用场景

在有限的情形下可以使用volatile代替锁。要想使volatile变量提供理想的线程安全，必须同时满足下面两个条件：

1.**对变量的写操作不依赖于当前值**。

2.**该变量没有包含在具有其他变量的不变式中**。

volatile最适用一个线程写，多个线程读的场景。

### synchronized

当它用来修饰一个方法或者一个代码块的时候，能够保证在同一时刻最多只有一个线程执行该段代码。

1.当两个并发线程访问同一个对象中的这个synchronized(this)代码块的时候，一个时间内只有一个线程能执行，另一个线程必须等待当前线程执行完这个代码块以后才能执行该代码块。

2.然而，当一个线程访问object的一个synchronized(this)同步代码块时，另一个线程仍然可以访问该object中的非synchronized(this)同步代码块。

3.尤其关键的是，当一个线程访问object的一个synchronized(this)同步代码块时，其他线程对object中所有其它synchronized(this)同步代码块的访问将被阻塞。

4.当一个线程访问object的一个synchronized(this)同步代码块时，它就获得了这个object的对象锁。结果，其它线程对该object对象所有同步代码部分的访问都被暂时阻塞。

### 区别

1.volatile是变量修饰符，而synchronized则是作用于一段代码或方法。

2.volatile只是在线程内存和“主”内存间同步某个变量的值；而synchronized通过锁定和解锁某个监视器同步所有变量的值, 显然synchronized要比volatile消耗更多资源。

3.volatile保证数据的可见性，但不能保证原子性；而synchronized可以保证原子性，也可以间接保证可见性，因为它会将私有内存中和公共内存中的数据做同步。

4.volatile标记的变量不会被编译器优化；synchronized标记的变量可以被编译器优化。（及volatile可以防止指令重排序，而synchronized也不会）。

