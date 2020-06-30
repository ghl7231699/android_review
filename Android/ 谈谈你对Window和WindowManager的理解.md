# 20191220
### 谈谈你对Window和WindowManager的理解

有时候我们需要在桌面上显示一个类似悬浮窗的东西，这种效果就需要用 Window 来实现，Window 是一个抽象类，表示一个窗口，它的具体实现类是 PhoneWindow，实现位于 WindowManagerService 中。

#### WindowManagerService
WindowManagerService 就是位于 Framework 层的窗口管理服务，它的职责就是管理系统中的所有窗口。窗口的本质是什么呢？其实就是一块显示区域，在 Android 中就是绘制的画布：Surface，当一块 Surface 显示在屏幕上时，就是用户所看到的窗口了。WindowManagerService 添加一个窗口的过程，其实就是 WindowManagerService 为其分配一块 Surface 的过程，一块块的 Surface 在 WindowManagerService 的管理下有序的排列在屏幕上，Android 才得以呈现出多姿多彩的界面。于是根据对 Surface 的操作类型可以将 Android 的显示系统分为三个层次，如下图：

