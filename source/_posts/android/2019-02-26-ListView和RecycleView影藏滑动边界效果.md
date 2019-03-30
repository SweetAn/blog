---
title: ListView和RecycleView影藏滑动边界效果
date: 2018-9-15 22:13:11
tags: 
- Android控件
categories: 
- Android
- 控件
---

### 一、首先是Listview的属性设置
设置滑动到顶部和底部的背景或颜色：

```
android:overScrollFooter="@android:color/transparent"
android:overScrollHeader="@android:color/transparent"
```

设置滑动到边缘时无效果模式：

```
android:overScrollMode="never"
```
设置滚动条不显示：

```
android:scrollbars="none"
```
以下是整体设置（overScrollHeader和overScrollFooter可不写，此处写了是引用的透明色）

``` xml
<ListView
    android:id="@+id/lv_type"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:overScrollFooter="@android:color/transparent"
    android:overScrollHeader="@android:color/transparent"
    android:overScrollMode="never"
    android:scrollbars="none">
```
二、RecyclerView的属性设置
以下是整体设置：

``` xml
<android.support.v7.widget.RecyclerView
    android:id="@+id/rv_search_one"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:overScrollMode="never"
    android:scrollbars="none" />
```




