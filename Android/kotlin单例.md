#Kotlin中实现单例的几种常见方式？

对比下Java中常见的5种单例模式和对应的转换成Kotlin后的。

### 饿汉式
#### java
```
class Singleton{
	private static Singleton instance=new Singleton();
	
	private Singleton(){
	
	}
	public static getInstance(){
	return instance;
	}
}
```
#### kotlin

```
class Singleton {
    //饿汉式
    val instance: Singleton = Singleton()
}
```

### 懒汉式(线程不安全)
#### java
```
public class Singleton {
    private static Singleton instance;
    private Singleton(){
 
    }
    public static Singleton getInstance(){
        if(instance==null){
            instance=new Singleton();
        }
        return instance;
    }
}
```
#### kotlin
```
class Singleton {
    var instance: Singleton? = null
        get() {
            if (field == null) {
                field = Singleton()
            }
            return field
        }
}
```

### 懒汉式（线程安全）
#### java

```
public class Singleton{
   //懒汉式 线程安全
    private static Singleton instance;

    private Singleton() {

    }

    public static synchronized Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}

```
#### kotlin
```
//懒汉式 线程安全
class Singleton {
    @get:Synchronized
    var instance: Singleton? = null
        get() {
            if (field == null) {
                field = Singleton()
            }
            return field
        }
        private set
}
```
### 静态内部类
#### java
```
    // 静态内部类
    private static class SingletonHolder {
        private static Singleton instance = new Singleton();

    }

    private Singleton() {
    }

    public static Singleton getInstance() {
        return SingletonHolder.instance;
    }
```
#### kotlin
```
class Singleton {
    val instance: Singleton
        get() = SingletonHolder.instance

    //静态内部类
    private object SingletonHolder {
        internal val instance: Singleton = Singleton()
    }
}
```
### 双重校验锁（通常线程安全，低概率不安全）
#### java

```
public class Singleton {
    private static Singleton instance;
    private Singleton(){
       
    }
    public static Singleton getInstance(){
        if(instance==null){
            synchronized (Singleton){
                if(instance==null){
                    instance=new Singleton();
                }
            }
        }
        return instance;
    }
}

```
#### kotlin
```
class Singleton {
    //双重校验锁
    var instance: Singleton? = null
        get() {
            if (field == null) {
                synchronized(Singleton::class.java) {
                    if (field == null) {
                        field = Singleton()
                    }
                }
            }
            return field
        }
        private set
}
```
还有一种很多文章都称之为实现单例的最完美方法：**枚举单例**

```
   //枚举方法（线程安全）
    enum SingletonDemo{
        INSTANCE;
        public void otherMethods(){
            System.out.println("Something");
        }
    }
```
这种方法写法超级简单，而且也能解决大部分的情况。但是如果在需要继承的情境下，这种写法就不适用了。

```
    internal enum class SingletonDemo {
        INSTANCE;

        fun otherMethods() {
            println("Something")
        }
    }
```
