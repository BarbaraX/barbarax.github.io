---
layout: post
title: "Android UI优化"
date: 2016-10-18
categories: Android
tags: Android UI
author: BarbaraX
---
* content
{:toc}

​	事情的起因是这样的，前几天同事打开了手机Developer Options上的Show GPU Overdraw开关。这样在页面上就可以看到用不同颜色区分出来的页面不同程度的过度绘制情况。有一个我写的页面存在很严重的过度绘制问题。于是我便趁机学习了一下Android上的UI优化知识点。

​	Android每16ms就会去渲染一次UI，如果某个UI的渲染操作超过16ms，就会存在失帧的情况。也就是我们平时说的卡顿。为了让UI渲染的时间尽量短，主要应该考虑一下两点：

1. 布局中View的嵌套层级尽量浅
2. 尽量避免设置无用的背景和背景颜色，也就是尽量避免在一个帧内多次绘制同一个像素点




#### 辅助工具

* __Hierarchy Viewer__

用来查看view嵌套层级的adb工具。它只能在开发者版本的机器上使用。步骤如下：

1. 终端中cd到sdk目录的tools目录下，输入`hierarchyviewer`
2. 默认会选中当前页面，点击Load View Hierarchy，就可以查看页面view tree了
3. 可以比较方便的看到页面哪些View是多余的，可以想办法去除的
4. 也可以方便的看到是页面的嵌套大概有多深
5. Hierarchy Viewer还有一个功能，是可以显示一个红绿灯，分别标注页面的measure， layout， draw是否需要优化的。点击页面还可以具体看到这三个过程的耗时

* __[Show GPU Overdraw](https://developer.android.com/studio/profile/dev-options-overdraw.html)__

用来查看重复绘制的debug工具。深红色的区域为重度绘制区域，需要被优化。这里需要注意的是，view的嵌套深度并不会影响过度绘制。一般都是多次设置了一个区域的背景导致的过度绘制，可以通过取消不会显示的view的背景来改善。

#### 优化方法

###### `<include layout = "@layout/xx">`复用，提高开发速度

```xml
<RelativeLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/activity_main"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:paddingBottom="@dimen/activity_vertical_margin"
    android:paddingLeft="@dimen/activity_horizontal_margin"
    android:paddingRight="@dimen/activity_horizontal_margin"
    android:paddingTop="@dimen/activity_vertical_margin"
    tools:context="com.example.xiongzhanyun.uiimprove.MainActivity">

    <include layout="@layout/common_header"/>

</RelativeLayout>
```

​	布局文件**common_header.xml**定义的是一个应用中的通用header。这个布局会在应用的多个页面中用到。通过把这种通用的布局include到各个页面中，实现了布局的复用。当需要改样式的时候，只需要改这个通用的xml文件，就可以作用到所有include的布局中，开发效率也是蹭蹭蹭的。

​	`<include>`的布局可以设置其他属性，比如id，这样就可以用新的id来覆盖原来xml文件的根布局id了。但是如果想要设置其他的属性，必须要先设置`layout_with`和`layout_height`这两个属性。

###### `<merge> </merge>`去掉重复布局

作用：

用来降低根布局是`FrameLayout`的层级。因为Android Activity的布局是包住一个`FrameLayout`当中的。

当`<include>`的布局的根与父布局的相同时

```xml
<merge
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/activity_main"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:paddingBottom="@dimen/activity_vertical_margin"
    android:paddingLeft="@dimen/activity_horizontal_margin"
    android:paddingRight="@dimen/activity_horizontal_margin"
    android:paddingTop="@dimen/activity_vertical_margin"
    tools:context="com.example.xiongzhanyun.uiimprove.MainActivity">

    <include layout="@layout/common_header"/>
</merge>
```

比方说，**common_header.xml**文件的最外层布局为`RelativeLayout`时，被include到一个根布局也是`RelativeLayout`的布局中，就可以用`merge`标签来替代父布局的`RelativeLayout`。可以在hierarchy viewer中看到，相同的布局被merge了！这样就少了一层嵌套深度~

###### `<ViewStub />`懒加载

只有真正需求的时候才会去创建view对象，不会在初始化的时候做。对于下载进度条，错误提示这种不是一直显示在页面上的布局就可以用ViewStub来懒加载。

```xml
<merge
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/activity_main"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:paddingBottom="@dimen/activity_vertical_margin"
    android:paddingLeft="@dimen/activity_horizontal_margin"
    android:paddingRight="@dimen/activity_horizontal_margin"
    android:paddingTop="@dimen/activity_vertical_margin"
    tools:context="com.example.xiongzhanyun.uiimprove.MainActivity">

    <include android:id="@+id/header"
        layout="@layout/common_header"/>

    <ViewStub android:layout_width="wrap_content"
              android:layout_height="wrap_content"
        android:layout="@layout/layout_viewstub"
        android:layout_below="@id/header"/>

</merge>
```

用法就是这样！当需要用到布局说，只需要inflate或者是setVisibility即可

```java
        //显示viewStub
        //viewStub.inflate();
        viewStub.setVisibility(View.VISIBLE);
```

问题TBD：

- 只能引入布局文件么？有的时候会导致嵌套层数变多啊

###### view尽量少设置background，降低Overdraw

​	对于一些在最下层不会显示的根布局，就不要设置背景颜色啦，或者在selector的normal情况下设置为transparent。



#### 参考

1. http://hukai.me/android-performance-patterns/
2. http://gold.xitu.io/entry/578b46b375c4cd5048506bc7
