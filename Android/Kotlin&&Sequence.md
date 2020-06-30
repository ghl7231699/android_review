### 谈谈Kotlin中的Sequence，为什么它处理集合操作更加高效？

[官网Sequence](https://kotlinlang.org/docs/reference/sequences.html#sequences)

我们先了解下Sequence。

举个栗子：

```
listof(1,2,3,4)
	.asSequence()
	.map{ it * it }
	.find{ it  > 3 }
	
//输出结果为：4
```

![](https://note.youdao.com/yws/api/personal/file/WEBa485adfb8b48365f30660172f679c6d1?method=download&shareKey=cd048ce9c4bf29da1b3805511cec9d0a)

通过官方文档，我们知道Sequence是个惰性集合
如图：

List处理数据时，每一个操作都会运用到所有元素，且**生成新的List**并继续下一步操作(感觉是横向的)；

Sequence处理数据时，针对每一个元素，执行所有操作流，直接得出单个元素的最终结果（感觉是纵向的）；

我们看下官方的例子：

#### Iterable
```
val words = "The quick brown fox jumps over the lazy dog".split(" ")
val lengthsList = words.filter { println("filter: $it"); it.length > 3 }
    .map { println("length: ${it.length}"); it.length }
    .take(4)

println("Lengths of first 4 words longer than 3 chars:")
println(lengthsList)

```
运行结果为：

```
filter: The
filter: quick
filter: brown
filter: fox
filter: jumps
filter: over
filter: the
filter: lazy
filter: dog
length: 5
length: 5
length: 5
length: 4
length: 4
Lengths of first 4 words longer than 3 chars:
[5, 5, 5, 4]
```
运行上述代码时，你会看到filter()和map()函数的执行顺序与代码中显示的顺序相同。首先查看filter:所有元素，然后length，最后输出结果。整个的流程如下：

![](https://kotlinlang.org/assets/images/reference/sequences/list-processing.png)

#### Sequence
让我们用Sequence实现同样的功能：

```
val words = "The quick brown fox jumps over the lazy dog".split(" ")
//convert the List to a Sequence
val wordsSequence = words.asSequence()

val lengthsSequence = wordsSequence.filter { println("filter: $it"); it.length > 3 }
    .map { println("length: ${it.length}"); it.length }
    .take(4)

println("Lengths of first 4 words longer than 3 chars")
// terminal operation: obtaining the result as a List
println(lengthsSequence.toList())

```

运行结果为：

```
Lengths of first 4 words longer than 3 chars:
filter: The
filter: quick
length:5
filter: brown
length:5
filter: fox
filter: jumps
length:5
filter: over
length:4
filter: the
filter: lazy
length:4
[5, 5, 5, 4, 4]

```

emmm  卧槽，这个顺序感觉像是开了个子线程，但又不完全符合；

从输出结果我们可以看到，仅当构建结果列表时才调用filter()和map()函数。因此，您首先看到文本行，“Lengths of..”然后开始进行序列处理。

请注意，在过滤完当前元素之后，会在当前元素执行map之后才会去过滤下一个元素。当结果的数量达到4个时，就会停止处理，因为它是take(4)可能返回的最大大小。

**Sequence 处理的过程如下：**

![](https://kotlinlang.org/assets/images/reference/sequences/sequence-processing.png)


对过上述两种方式，我们发现Sequence明显减少了循环次数

Sequences提高性能的秘密在于这三个操作可以共享同一个迭代器(iterator)，只需要一次循环即可完成。Sequences允许 map 转换一个元素后，立马将这个元素传递给 filter操作 ，而不是像集合(lists) 那样，等待所有的元素都循环完成了map操作后，用一个新的集合存储起来，然后又遍历循环从新的集合取出元素完成filter操作。

### 扩展  Java8的Stream(流)

```
list.asSequence()
    .filter { it < 0}
    .map { it++ }
    .average()

list.stream()
    .filter { it < 0}
    .map { it++ }
    .average()

```

上面的代码大家可以运行一下。

stream的处理效率几乎和Sequences 一样高。它们也都是基于惰性求值的原理并且在最后(终端)处理集合。

