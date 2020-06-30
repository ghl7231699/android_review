###谈谈Kotlin中的Unit？它和Java中的void有什么区别？
在Kotlin中是没有void的，所有函数都有返回类型。举个栗子：

```
fun doSomething(){
}
```
这个函数的返回类型是Unit，而且所有不显式生命返回类型的函数都会返回Unit类型。所以写不写Unit并没有什么区别。上面的代码可以写作：

```
fun doSomething():Unit{

}
```
因为Unit是一个真正的返回类型，所以我们可以将Unit类型的值赋值给一个变量。

```
val result=donSomething()
```
那么Unit和Java中的void有什么区别呢？

##### 同
两者概念相似
##### 异

1. Unit是一个真正的类，继承自Any类，只有一个值，也就是所谓的单例（目的在于函数返回Unit时避免分配内存）。
2. 因为Unit是一个普通的对象（这里指用 object 关键字定义的单例类型），所以可以调用它的toString()方法：结果一定是“Kotlin.Unit”，因为在源码中已经写死了。

```
public object Unit {
    override fun toString() = "kotlin.Unit"
}
```

翻译自

[Nothing (else) matters in Kotlin](https://proandroiddev.com/nothing-else-matters-in-kotlin-994a9ef106fc)
