# 20191230
### 说说Kotlin中的Any与Java中的Object有何异同？

首先说下相同点：

* 都是顶级父类

不同点：

* 方法数不同

    Any中定义的方法有：toString()、equals()、hashCode3个。
    
    Object类中定义的方法有：toString()、equals()、hashCode()、getClass()、clone()、finalize()、notify()、notifyAll()、wait()、wait(long)、wait(long,int)  11个。

关联：

Kotlin编译器将Kotlin.Any和java.lang.Object类视作两个不同的类，但是在运行时它们俩就是一样的。如果你打印print(Any().javaClass)会发现打印出来的结果是class java.lang.Object。