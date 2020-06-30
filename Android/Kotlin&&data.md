### 谈谈你对Kotlin中的data关键字的理解？相比于普通类有哪些特点？

[官网地址](http://www.kotlincn.net/docs/reference/data-classes.html#data-classes)

直接看官网。

我们经常创建一些只保存数据的类。 在这些类中，一些标准函数往往是从数据机械推导而来的。在 Kotlin 中，这叫做 数据类 并标记为 data：

```
data class User(val name:String,val age:Int)
```

编译器自动从主构造函数中声明的所有属性导出以下成员：

1. equals()/hashCode() 对；
2. toString() 格式是 "User(name=John, age=42)"；
3. componentN() 函数 按声明顺序对应于所有属性；
4. copy()。

为了确保生成代码的一致性以及有意义的行为，数据类必须满足一下要求：

1. 主构造函数需要至少有一个参数；
2. 主构造函数的所有参数需要标记为val或var；
3. 数据类不能是抽象、开放、密封或者内部的**（abstract, open, sealed or inner）**；
4. （在1.1之前）数据类只能实现接口。

此外，成员生成遵循关于成员继承的这些规则：

1. 如果在数据类体中有显式实现 equals()、 hashCode() 或者 toString()，或者这些函数在父类中有 final 实现，那么不会生成这些函数，而会使用现有函数；
2. 从一个已具copy(.....)函数且签名匹配的类型派生出一个数据类在Kotli1.2中已弃用，并且在1.3中禁用；
3. 不允许为 componentN() 以及 copy() 函数提供显式实现。

在JVM中，如果生成的类需要含有一个无参的构造函数，则所有的属性必须制定默认值。

```
data class User(val name:String = "",val age:Int = 0)
```


#### 在类体中声明的属性

请注意，对于那些自动生成的函数，编译器只使用在主构造函数内部定义的属性。如需在生成的实现中排除一个属性，请将其声明在类体中：

```
data class Man(val name: String) {
    var age: Int = 0
}

fun main(){
  val man1 = Man("lili")
    val man2 = Man("lili")

    man1.age = 10
    man2.age = 20

    println("man1 == man2=${man1 == man2}")
}
```

最后的输出结果为：

```
man1 == man2=true
```

在toString()、equals()、hashCode()、以及copy()的实现中，只会用到name属性，并且只有一个component函数component1()。在上面的例子中，虽然两个Man对象的年龄不同，但是会被视为相等。

#### Copying

在很多情况下，我们需要复制一个对象改变它的一些属性，但其余部分保持不变。 copy() 函数就是为此而生成。对于上文的 User 类，其实现会类似下面这样：

```
fun copy(name: String = this.name, age: Int = this.age) = User(name, age)
```

这让我们可以写：

```
val jack = User(name = "Jack", age = 1)
val olderJack = jack.copy(age = 2)
```

#### 数据类与解构声明

为数据类生成的Component函数，使他们可在解构声明中使用：

```
val jane = User("Jane", 35)
val (name, age) = jane
println("$name, $age years of age") // 输出 "Jane, 35 years of age"
```

#### 标准数据类

标准库提供了 Pair 与 Triple。尽管在很多情况下命名数据类是更好的设计选择， 因为它们通过为属性提供有意义的名称使代码更具可读性。