# 20191212

### 谈下Android签名校验机制，v1.v2.v3各有什么不同？为什么会有这样的发展趋势？

### Android-APK签名工具-jarsigner和apksigner

```
jarsigner是JDK提供的针对jar包签名的通用工具，位于JDK/bin/下

apksigner是Google官方提供的针对Android apk签名及验证的专用工具，
位于Android SDK/build-tools/SDK版本/apksigner。使用pk8、x509.pem文件进行签名。其中pk8是私钥文件，x509.pem是含有公钥的文件。生成的签名文件统一使用“CERT”命名。

不管是apk包，还是jar包，本质都是zip格式的压缩包，所以它们的签名过程都差不多（仅限V1签名）

这两个工具都可以对Android apk包进行签名
```
### V1和V2签名的区别

```
在Android Studio中点击菜单 Build->Generate signed apk ...打包签名中，可以看到V1（jarsignature）和V2（Full APK Signature）两种签名方式。

从Android7.0开始，Google增加新签名方案V2 Scheme（APK signature）,但7.0以下版本只能用旧的签名方案V1Scheme（JAR Signature）

V1签名:
    来自JDK(jarsigner), 对zip压缩包的每个文件进行验证, 签名后还能对压缩包修改(移动/重新压缩文件)
    对V1签名的apk/jar解压,在META-INF存放签名文件(MANIFEST.MF, CERT.SF, CERT.RSA), 
    其中MANIFEST.MF文件保存所有文件的SHA1指纹(除了META-INF文件), 由此可知: V1签名是对压缩包中单个文件签名验证

V2签名:
    来自Google(apksigner), 对zip压缩包的整个文件验证, 签名后不能修改压缩包(包括zipalign),
    对V2签名的apk解压,没有发现签名文件,重新压缩后V2签名就失效, 由此可知: V2签名是对整个APK签名验证
    V2签名优点很明显:
        签名更安全(不能修改压缩包)
        签名验证时间更短(不需要解压验证),因而安装速度加快

注意: apksigner工具默认同时使用V1和V2签名,以兼容Android 7.0以下版本
```
### V3

Android9.0中引入了新的签名方式，它的格式大体和V2类似，在V2插入的签名块中，添加了一个新块（Attr块）。

在这个新块中，会记录我们之前的签名信息以及新的签名信息，以密钥转轮的方案，来做签名的替换和升级。这意味着，只要旧签名证书在手，我们就可以通过它在新的 APK 文件中，更改签名。

v3 签名新增的新块（attr）存储了所有的签名信息，由更小的 Level 块，以链表的形式存储。

其中每个节点都包含用于为之前版本的应用签名的签名证书，最旧的签名证书对应根节点，系统会让每个节点中的证书为列表中下一个证书签名，从而为每个新密钥提供证据来证明它应该像旧密钥一样可信。

这个过程有点类似 CA 证书的证明过程，已安装的 App 的旧签名，确保覆盖安装的 APK 的新签名正确，将信任传递下去。

关于V3可以参考官方文档[https://source.android.com/security/apksigning/v3](https://source.android.com/security/apksigning/v3)

搜到一篇全面的讲述android签名的文章[https://github.com/jeanboydev/Android-ReadTheFuckingSourceCode/blob/master/article/android/basic/09_signature.md](https://github.com/jeanboydev/Android-ReadTheFuckingSourceCode/blob/master/article/android/basic/09_signature.md)
