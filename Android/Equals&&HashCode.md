### 20191108
#### equals vs hashcode
* 两个对象，equals相同，hashcode一定相同吗？
* hashcode相同，equals一定相同吗？
* 二者一般在什么时候配合？如何配合？在哪些源码中有类似的配合。  

针对前两个问题，答案很简单：equals相同，hashCode一定相同；hashCode相同，equals不一定相同。

以下这段 Object 的 equals()方法注释，也可以表明这个事实：


```
Note that it is generally necessary to override the {@code hashCode}
     * method whenever this method is overridden, so as to maintain the
     * general contract for the {@code hashCode} method, which states
     * that equal objects must have equal hash codes.
```

针对第二个问题，因为 hashcode 是 int 值，取值是有限的，而要存放的值是无限的，因此肯定会存在冲突，这也叫鸽巢原理。因此不同对象的 hashcode 可能相同
即 hashcode 相同， equles 不一定为 true
以下这段 Object 的 hashCode() 方法注释，也可以表明这个事实：

```
<p>
     * <li>If two objects are equal according to the {@code equals(Object)}
     *     method, then calling the {@code hashCode} method on each of
     *     the two objects must produce the same integer result.
     * <li>It is <em>not</em> required that if two objects are unequal
     *     according to the {@link java.lang.Object#equals(java.lang.Object)}
     *     method, then calling the {@code hashCode} method on each of the
     *     two objects must produce distinct integer results.  However, the
     *     programmer should be aware that producing distinct integer results
     *     for unequal objects may improve the performance of hash tables.
     * </ul>
     * <p>

```

至于两者的配合使用，在HashMap、HashTable源码中就有对应的配合。HashMap中的put方法就是先查看hashcode值，如果存在hashcode就调用equals查看是否存在，存在则更新，没有则插入新值，在插入大量非重复的数据时，可以有效的减少equals的调用，从而提高效率。

#### 知识补充局
* 为什么重写hashCode方法时，使用31？

1. 质数产生哈希冲突的几率小，31 ，41，43 都可以

2. 质数越大，冲突几率越小，但性能越差，31 是折中的选择

3. jvm 对 31 的 has 运算有优化

