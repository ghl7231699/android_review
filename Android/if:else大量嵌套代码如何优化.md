### 20191126
#### 对于代码中有大量的 if/else 你有什么优化思路？

经验略谈。

1.策略模式：将if代码块的方法根据需要封装成类的action方法，然后用工厂模式创建对应对象，使用对应方法。[传送门](https://www.runoob.com/design-pattern/strategy-pattern.html)

2.类似于事件传递机制一样，采用责任链模式层层传递。[传送门](https://www.runoob.com/design-pattern/chain-of-responsibility-pattern.html)

3.else代码放在函数前面，尽早返回。

4.使用Map替代分支语句。

5.部分的if else 可封装成方法。

策略模式&责任链模式  优缺点可以参考对应的链接。

开放式题目，纯属经验之谈，还是要考虑实际运用场景来选择合适的方式。