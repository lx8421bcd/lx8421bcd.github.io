---
layout:     post
title:      TV端开发之RecyclerView
subtitle:   Android TV端开发从0到1的一点经验
date:       2018-12-09
author:     lx8421bcd
header-img: img/post-bg-debug.png
catalog: true
tags:
    - Android
    - Android TV端
---

### 前言
RecyclerView自2014年Google推出以来，就因为其低耦合、灵活、高可扩展性而广受欢迎，在我自己开发过的项目中基本已经完全摒弃ListVie和GridView等控件，全面使用RecyclerView，开发AndroidTV端同样不例外，使用RecyclerView带来的便利性毋庸置疑，但Google在开发控件时似乎并没有考虑到TV端，直接在TV端使用RecyclerView，会有一些小问题需要处理。


### 使用注意
在TV端使用RecyclerView与手机端并没有多大不同，不过有一些问题需要注意。  

在pre5.0系统上，ViewGroup、TextView、ImageView等等这些“看起来不应该有交互”的控件，默认是拿不到焦点的，给它设置```OnFocusChangeListener```并不会改变什么，__必须```setFocusable(true);```或者在xml配置中添加 ```android:focusable="true"```，这些控件才会正确的被系统派发焦点__。这一点需要特别注意。


### 放大遮盖问题处理
此处实现请参考文章[TV端开发之选中放大效果实现](https://lx8421bcd.github.io/2018/12/07/TV%E7%AB%AF%E5%BC%80%E5%8F%91%E4%B9%8B%E9%80%89%E4%B8%AD%E6%94%BE%E5%A4%A7%E6%95%88%E6%9E%9C%E5%AE%9E%E7%8E%B0/)


### 滚动居中效果
此处实现请参考文章 [TV端开发之焦点控件垂直居中](https://lx8421bcd.github.io/2018/12/08/TV%E7%AB%AF%E5%BC%80%E5%8F%91%E4%B9%8B%E7%84%A6%E7%82%B9%E6%8E%A7%E4%BB%B6%E5%9E%82%E7%9B%B4%E5%B1%85%E4%B8%AD/)

### 刷新数据焦点丢失问题
由于TV端焦点控制的存在，因此在使用RecyclerView时要尽可能避免直接```notifyDataSetChanged()```进行全局刷新（ps:RecyclerView本身就不推荐这种方法刷新），尽可能的去使用RecyclerView的局部刷新方法，避免分页加载数据时造成焦点丢失、滚动错乱等问题。

### 焦点乱飞问题
这一点应该算是在TV端使用RecyclerView最头疼的问题了，主要出现在遥控器长按，连续快速按键等操作时。  
我研究了一下RecyclerView和LayoutManager的源码，发现出一点端倪：  

在滚动加载数据时，焦点先是由RecyclerView底部的item拿到，这时下一分页数据过来，系统使用被缓存的ViewHolder填充数据，这时进入下一组数据的加载阶段，如果焦点飞到这些正在加载数据的View中，将会直接丢失焦点。  
使用```GlobalFocusChangeListener```监听这个阶段的焦点变化回调就会发现，当焦点飞到其它地方去的时候，上一个焦点View为null；也就是说原因跟notifyDataSetChanged类似，都是在加载数据的过程中丢失焦点。

__那么，怎么处理？__  
目前网上讨论的解决方案由两种，一个是自定义LayoutManager，在触底时停止；一个是自定义RecyclerView，修改其```dispatchKeyEvent()```和```onScrolled()```方法[FocusRecyclerView](https://github.com/genius158/TVProjectUtils/blob/master/tvprojectutils/src/main/java/com/yan/tvprojectutils/FocusRecyclerView.java)。这两种方法在实际使用上都有缺陷，修改LayoutManager的边界判断逻辑有时会导致焦点无法跳出RecyclerView，而修改RecyclerView的方法并没有完全解决问题，焦点乱跳依然存在，而且由于其直接使用了FocusFinder而不是交由父View搜索焦点，焦点在View间跳转时会出现不合逻辑的焦点派发。

__目前我发现的改动成本最低，效果最好的手段，是限制按键输入频率__

```java
// code in TVRecyclerView.java (my custom RecyclerView)

    private long mLastKeyDownTime;

    @Override
    public boolean dispatchKeyEvent(KeyEvent event) {
        long current = System.currentTimeMillis();
        if (event.getAction() != KeyEvent.ACTION_DOWN || getChildCount() == 0) {
            return super.dispatchKeyEvent(event);
        }
        // 限制两个KEY_DOWN事件的最低间隔为120ms
        if (isComputingLayout() || current - mLastKeyDownTime <= 120) {
            return true;
        }
        mLastKeyDownTime = current;
        return super.dispatchKeyEvent(event);
    }
```
在我的项目中测试，120ms是基本不会出现飞焦点，同是滑动速度也比较快的限制值。这一点可以自己调整，也可以写成可配置参数。  
为了测试这个问题，写文章时还专门下了个斗鱼TV客户端，发现斗鱼的直播间列表居然直接把长按事件屏蔽了，emmmm......   

### support-v17
Google专门为AndroidTV端开发提供的android-support-v17库中包含了不少RecyclerView的封装，主要为[BaseGridView](https://developer.android.com/reference/android/support/v17/leanback/widget/BaseGridView)及其子类[HorizontalGridView](https://developer.android.com/reference/android/support/v17/leanback/widget/HorizontalGridView)和[VerticalGridView](https://developer.android.com/reference/android/support/v17/leanback/widget/VerticalGridView)。  
由于support-v17库的版本要求，我们并不方便直接将整个库完全引用，如果需要使用Google为TV端编写的RecyclerView控件，可以考虑源码集成。