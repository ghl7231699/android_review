# 20191224
### Kotlin中怎么给系统中的类，动态添加的方法？
Kotlin中比较吸引人的特性就是：扩展方法。

可以在不修改类代码的情况下，动态为类添加方法。

比如说：

```
12.dp() //用于将整形px转化为dp值，dp()是动态添加的方法
```

那么问题来了：
1. 它是怎么做到的
2. 可以利用这个特性“覆盖”掉某个类的已有方法吗
3. 这个特性有什么约束


#### 拓展方法是怎么做到的
我们先自己定义一个：

```
fun String.second() = this[1]
```
通过AS我们看下反编译后的Java代码：

```
public static final char second(@NotNull String $this$second) {
      Intrinsics.checkParameterIsNotNull($this$second, "$this$second");
      return $this$second.charAt(1);
   }
```

我们可以看到，我们定义的扩展方法second,最终会变成一个静态的方法，方法的参数就是目标类对象.

接下来我们调用这个方法：

```
fun main() {
    println("123".second())
}
```
查看反编译之后的：

```
public static final void main() {
      char var0 = second("123");
      System.out.println(var0);
   }

   // $FF: synthetic method
   public static void main(String[] var0) {
      main();
   }
```

emm.......

调用拓展方法的地方，转成Java后跟调用静态方法没有区别，只是编译器会帮助我们传了参数。

#### 拓展方法能覆盖已有的方法吗
答案是否定的。

如果一个类的扩展方法跟已有的方法同名（签名），那么在调用的时候，会优先调用类中的方法，并不会调用扩展方法，即使手动import。

因为扩展方法跟现有方法同名(同签名)的话，在编译成class时，编译器并不知道你本次的调用究竟是想调用哪一个(只能当成是调用已有的方法)，所以想要扩展方法生效，就不能跟目标类的现有方法重名(同签名)。

#### 有什么约束

同一级包下不能有多个相同签名的方法；

包名不同，签名相同的扩展方法，可以通过导入指定的包来告诉编译器想使用哪个方法；