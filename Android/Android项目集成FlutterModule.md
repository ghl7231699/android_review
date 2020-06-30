## Android项目集成Flutter Module

在android项目中，需要以flutter module的形式来集成。

### 创建flutter module

主要有两种方式：手动创建和IDE创建

### 手动创建

在项目的根目录下，执行以下命令新建一个flutter module

```
flutter create -t module flutter_mall(module名称)
```

#### 集成
集成的方式有两种。

##### 手动集成

先进入到新建的module下，执行以下命令行

```
cd flutter_mall/
cd .android/
./gradlew flutter:assembleDebug

```
同时，我们需要在项目的build.gradle中配置java8的兼容：

```
android{
	
	//flutter相关声明
    compileOptions {
        sourceCompatibility 1.8
        targetCompatibility 1.8
    }

}

```
在根项目的setting.gradle中配置：

```
setBinding(new Binding([gradle: this]))
evaluate(new File(
        rootDir.path + '/flutter_mall/.android/include_flutter.groovy'
))

```
最后在app的build.gradle中配置依赖：

```
dependencies {
	implementation project(':flutter')
}
```

##### IDE 依赖自动导入

直接在AS中 File->new Module->import Flutter Module  

一路next，最后同步。

### IDE

在AS中  File->new Module->Flutter Module  

### 问题

##### 1. 如果编译的过程中遇到

```
A problem occurred configuring root project 'android_road'.
> A problem occurred configuring project ':flutter'.
   > Failed to notify project evaluation listener.
      > assert appProject != null
               |          |
               null       false
```
这种异常的话，可以查看下自己的项目是否对app module名称进行了修改。因为在**flutter/packages/flutter_tools/gradle/flutter.gradle**中，会默认寻找“:app”module。

```
 Project appProject = project.rootProject.findProject(':app')
 assert appProject != null
```

##### 2. 因为Flutter项目中使用的是androidX，所以如果想集成flutter module,你的原生项目需要迁移到androidx。

##### 3. Module里不要fultter clean  否则会清空.android .ios .data_tools 文件，导致编译异常