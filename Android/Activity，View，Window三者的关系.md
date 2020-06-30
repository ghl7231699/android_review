# 20191218
### 谈一谈Activity，View，Window三者的关系

先说下Activity是如何展现在手机上的。

Activity本身并不负责视图控制，它只是控制生命周期和处理事件，真正控制视图的是Window。

Activity就像一个控制器，统筹视图的添加与显示，以及通过其他回调方法，来与Window、View进行交互。

1. 在Activity创建时调用attach方法，创建window;
2. attach方法中会生成phoneWindow。在Activity中调用setContentView()方法其实就是调用getWindow.setContentView()方法创建parentView，将指定的.xml布局文件进行填充。

三者关系：

1、在Activity中调用attach，创建了一个Window

2、创建的window是其子类PhoneWindow，在attach中创建PhoneWindow

3、在Activity中调用setContentView(R.layout.xxx)

4、其中实际上是调用的getWindow().setContentView()

5、调用PhoneWindow中的setContentView方法

6、创建ParentView：

       作为ViewGroup的子类，实际是创建的DecorView(作为FramLayout的子类）

7、将指定的R.layout.xxx进行填充

通过布局填充器进行填充【其中的parent指的就是DecorView】

8、调用到ViewGroup

9、调用ViewGroup的removeAllView()，先将所有的view移除掉

10、添加新的view：addView()