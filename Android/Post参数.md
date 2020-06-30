## Post中请求参数放在了哪个位置？

我们都知道get请求的话，是把参数包含在了Url当中，而post通过request body来传递参数（当然post请求也可以拼接在url当中进行传参）。

接下来我们讨论下GET、POST。

### HTTP的方法
**Get和Post的区别：**

我们讨论到Get方法和Post的区别时，大部分搜索给出的答案都是

1. Get的参数在URL中，Post的参数在HTTP的报文体Body里

2. Get参数有大小限制，Post没有大小限制

3. Get参数不安全，Post安全

以上答案是错误的参考文章[99%的人理解错Get和Post的区别](https://learnku.com/articles/25881)讲解为什么错误，我们这里只说结论

1. HTTP用什么请求和参数在哪里一点关系没有

2. HTTP协议对参数长度也没限制，大多数和服务器容器的配置有关

3. HTTP用什么方法都不安全，除非用HTTPS

### HTTP的参数

HTTP不管使用什么方法和参数放的位置都没有关系的。这里来介绍有几种放HTTP参数的方式。

#### URL里放参数

在Url中放参数最简单，就是?+键值对，它存在于HTTP的Header中第一行

```
POST /psas/bug/image/confirm?param1=123&param2=bbq HTTP/1.1
```

上面的包含两个参数：

参数 | 值 
:-: | :-: 
param1 | 123
param2 | bbq

由于URL里放参数是放在HTTP报文头，而往Body里放参数的方式就有很多种了，如何让接收放识别这些放参数的方法，就靠Content-type。

Body参数 | Content-type
:-: | :-:
Text | text/plain
Form | application/x-www-form-urlencoded
JSON | application/json
File | 不确定
Multipart | multipart/form-data; boundary=X_PAW_BOUNDARY

根据Content-type不同，服务器去读取HTTP Body中参数的方式也不一样。

#### Body不同参数方式介绍
##### text/plain 文本传输
一般这个。。很少用到

HTTP的报文体中是纯文本，没有任何格式和修饰，服务端就会拿走文本自己处理

##### application 参数传输
我们最常用到的就是 application 格式的HTTP报文体，其中还分为JSON格式与Form表单格式

##### Form 表单
这个是最常用的传递参数方式，HTML中都有form标签与其对应，其本身采用Key-Value的方式传递参数

其本身就是简单的把URL参数中？后的字符串移到Body里

所以看其Content-type全称 application/x-www-form-urlencoded 后边代表意思是 X-万维网-FORM表单-URL编码方式

##### JSON Object
与Form表单同属于一个Content-type分类JSON格式的全称是 application/json 与Form表单不同是，

他的HTTP的Body是一串符合JSON格式的字符串，而不是简单的把URL参数移动到Body内

所以说Json格式比Form更加有效的地方是可以传送Object，而不是简单的Key-Value对，但是还有聪明的小伙子用Form来发Object，参考Spring传JSON参数和文件

##### File 文件传输
这个也很少用，一般只在返回报文中出现，用于传输 单个文件 ，所以说根据传输的文件不同其Content-type也不同，例如

文件类型 | Content-type
:-: | :-:
png图片 | 	image/png
pdf文档	 | application/pdf

而HTTP的Body中则是文件的二进制数据

##### Multipart 复合传输

Multipart的中文直译是 “多个部分” 就是指的不仅可以传输参数(Value)还可以传输文件(File)，但是参数和文件之间怎么区分呢？

根据Content-type 除了使用 multipart/form-data 之外还有一个 boundary=X_PAW_BOUNDARY

其中前者代表 多个部分/表单数据 而后者代表boundary(边界)，是一个字符串 “X_PAW_BOUNDARY“，由于我用的软件是PAW，所以叫X_PAW,用浏览器可能是其它的字符串，然后在HTTP的Body中我们可以看到它

```
--__X_PAW_BOUNDARY__                               <==边界的开始
Content-Disposition: form-data; name="param1"      <==参数名
                                                   <==换行必须有
1                                                  <==参数值
--__X_PAW_BOUNDARY__                               <==第二个参数边界的开始
Content-Disposition: form-data; name="file"; filename="lover1.png"  <==第二个参数为文件 参数名file 文件名为lover1.png
Content-Type: image/png
                                                   <==换行
PNG                                                <==文件的数据信息


IHDRSJ®ïË	pHYs
......
......
--__X_PAW_BOUNDARY__                               <==第三个参数边界的开始
Content-Disposition: form-data; name="param2"      <==参数名
                                                   <==换行
b                                                  <==参数值
--__X_PAW_BOUNDARY__--                             <==所有参数的边界结束

```
由上文可以发现，HTTP的Body中使用两个短横线”–”加上boundary字符串作为不同参数的分割，而且不管是值参数(Value)还是文件参数(File)在Boundary内部都有自己的描述信息，并不是URL参数的简单移动

并且在结束的时候，不仅 前缀要加双短横线，后缀也要加，代表结束

### 结语

通常情况下x-www-form-urlencoded是最常用的传参方法

如果想传递Object可以使用JSON，使用SpringMVC的话需要特殊配置

上传文件用的最多的就是Multipart，Java有专门的Jar包来处理文件上传


## Tips

### getParameter()作用

看下该方法的官方说明：

```
Returns the value of a request parameter as a String, or null if the parameter does not exist. Request parameters are extra information sent with the request. For HTTP servlets, parameters are contained in the query string or posted form data.
```
通过上述描述，我们知道：getParameter()方法取的是query string 和 posted form data中的数据，那么query string 和 posted form data又是指的啥呢？

query string就是get请求中url的?后面的参数，posted form data就是post请求中的body数据。

平时我们用的都是ajax请求，无论是get还是post，我们的请求参数都是可以放在data里的，只不过ajax会判断请求类别，如果是get请求的话，ajax会把data里的值给拼接到url的后面。
