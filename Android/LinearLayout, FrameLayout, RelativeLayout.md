### 20191118
#### LinearLayout, FrameLayout, RelativeLayout 哪个效率高

这个问题需要针对不同的场景来进行说明。

先简单说下三种布局的概念：

LinearLayout:是一个方向性布局，通过方向上的计算来绘制布局。

FramLayout：是一个从屏幕左上角开始绘制的堆叠布局。

RelativeLayout：是一个根据控件的位置来确定位置的布局。

LinearLayout、RelativeLayout两者绘制同样的界面时layout和draw的过程时间消耗相差无几，关键在于onMeasure过程。

**RelativeLayout的onMeasure过程**

RelativeLayout会根据2次排列的结果对子view各做一次measure。

RelativeLayout中子View的排列方式是基于彼此的依赖关系，在确定每个子view的位置的时候，需要先给所有的child view排序一下，所以需要横、纵向两个方向分别进行一次measure。

**LinearLayout的onMeasure过程**

LinearLayout会先做一个方向的判断。在每次对child测量完毕后，都会调用child.getMeasureHeight/getMeasuredWidth()获取子视图的最终的高度，并将这个高度添加到mTotalLength中。

但是getMeasuredHeight暂时避开了lp.weight>0且高度为0子View，因为后面会将把剩余高度按weight分配给相应的子View。因此可以得出以下结论：

1.  如果在LinearLayout中不使用weight属性，将只进行一次measure的过程（如果使用weight属性，则遍历一次view测量后，再遍历一次view测量）。
2.  如果使用了weight属性，LinearLayout在第一次测量时获取所有子View的高度，之后再将剩余高度依据weight加到weight>0的子view上。


所以针对不同场景来说：

简单的布局：FrameLayout>LinearLayout>RelativeLayout

复杂的布局：FrameLayout<LinearLayout<RelativeLayout


布局最重要的还是减少层级嵌套才是王道啊！
