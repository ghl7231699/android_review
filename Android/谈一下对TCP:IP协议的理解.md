# 20200103
### 谈一下对TCP/IP协议的理解

#### 定义
即传输控制/网间协议，定义了主机如何连入因特网及数据如何在他们之间传输的标注。

TCP/IP协议是一个协议结合。为了叫的方便，所以大家统称为TCP/IP。在TCP
/IP协议中有一个重要的概念就是分层。TCP/IP协议参考模型把整个ISO模型归类到四个抽象层中：

应用层：TFTP，HTTP，SNMP，FTP，SMTP，DNS，Telnet 等等

传输层：TCP，UDP

网络层：IP，ICMP，OSPF，EIGRP，IGMP

数据链路层：SLIP，CSLIP，PPP，MTU

具体每层的职能如下图所示：

![](https://note.youdao.com/yws/api/personal/file/WEB7d3b611b847e34e7da32fdf7c4519f1d?method=download&shareKey=4ad3d09977b5d6b27beb0de9cd091f6c)

TCP/IP通信数据流如下：

![](https://note.youdao.com/yws/api/personal/file/WEBd90b8873e267a0b488dfffac62df619b?method=download&shareKey=00332d32c83137d79580cacc104b8501)

#### HTTP关系密切的协议：IP、TCP和DNS

**IP协议(Internet protocol)：**这里的IP不是我们通常所说的ip地址，而是指的一种协议。

IP协议的作用在于把各种数据包准确无误的传递给指定方，其中两个重要的条件是：ip地址和MAC地址。由于ip地址的总数是有限的（IPV4），不可能每个人都有一个ip地址，通常我们的ip地址都是路由器分配的，路由器里面会记录我们的MAC地址。MAC地址是惟一的（除去认为因素干预）。举一个现实生活中的例子，IP地址就如同是我们居住小区的地址，而MAC地址就是我们住的那栋楼那个房间那个人。

一下内容来自《图解Http》:

```
使用 ARP 协议凭借 MAC 地址进行通信

   IP 间的通信依赖 MAC 地址。在网络上，通信的双方在同一局域网（LAN）内的情况是很少的，通常是经过多台计算机和网络设备中转才能连接到对方。
   
   而在进行中转时，会利用下一站中转设备的 MAC 地址来搜索下一个中转目标。这时，会采用 ARP 协议（Address Resolution Protocol）。
   
   ARP 是一种用以解析地址的协议，根据通信方的 IP 地址就可以反查出对应的 MAC 地址
```

你向另外一台电脑发送一条信息，怎么在茫茫人海中找到对方，一下为图示：

![](https://note.youdao.com/yws/api/personal/file/WEB1771553beb017054f41c9310e21f665d?method=download&shareKey=552b59b77fee393901dad9010cb3015c)

**TCP协议（Transmission Control Protocol）：**如果说IP协议是找到对象详细的地址，那么TCP协议就是安全的把东西带给对方。

TCP属于传输层，提供可靠的字节流服务。字节流服务是什么呢？

所谓的字节流，类似于信息切割。比如说你是个卖车的，你要去送货。完整的车运送起来过于庞大，还容易损坏。那就将车拆成几个部分，每个部分都贴上收货人的地址。最后送到之后，再重新组装到一起，这个拆解、运输、拼装的过程其实就是TCP字节流的过程。

看下专业的学术表达：

```
所谓的字节流服务（Byte Stream Service）是指，为了方便传输，将大块数据分
割成以报文段（segment）为单位的数据包进行管理。而可靠的传输服务是指，能够
把数据准确可靠地传给对方。一言以蔽之，TCP 协议为了更容易传送大数据才把数据分割，而且 TCP 协议能够确认数据最终是否送达到对方。
```

为了确保信息能够准确无误的送达，TCP采用了著名的三次握手策略（threee way handshaking）。如下图示：

![](https://note.youdao.com/yws/api/personal/file/WEBe95c0658f1e05d6d89c619ac4b2725a8?method=download&shareKey=fba0a0abf4f24a4415b72d0034651796)

上面的图示是个简化的过程，大家感兴趣的话可以看下具体的实现。

**DNS(Domain names System)：**DNS和Http协议一样是处于应用层的服务，提供域名到IP地址之间的解析服务。

互联网之间是通过IP地址通信的，但是IP地址并不符合人们的记忆习惯，人们喜欢记忆有意义的词，比如说baidu.com。DNS就是为了解决这个问题而生的。比如说我们电脑中的host文件：

192.168.1.11   xiaozhu.com

当我们访问xiaozhu.com的时候，电脑便不会去外网服务器查询了，而直接去访问192.168.1.11 。这是一个简单的域名劫持，但足以说明DNS的含义了。如下图示意：

![](https://note.youdao.com/yws/api/personal/file/WEB46c574dac88152ceaa035d7ce11a813d?method=download&shareKey=d28a92025fe37f2c3a350eb8770c800f)



[参考文章传送门](https://www.cnblogs.com/laojiao/p/9653108.html)