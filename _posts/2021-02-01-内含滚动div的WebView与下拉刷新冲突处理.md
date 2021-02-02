---
layout:     post
title:      内含滚动div的WebView与下拉刷新冲突处理
subtitle:   
date:       2021-02-01
author:     lx8421bcd
header-img: img/post-bg-android.jpg
catalog: false
tags:
    - Android
---

给简单的WebView加下拉刷新布局很简单，直接RefreshLayout套在WebView外面就好了，比如谷歌的SwipeRefreshLayout。相当的简单粗暴。  
这种处理方法遇见大部分结构简单的长页面都是没有问题的。但假如页面结构比较复杂，就会吃瘪了，比如下面这个页面：

![Bilibili H5 首页](https://raw.githubusercontent.com/lx8421bcd/lx8421bcd.github.io/master/img/post_img/bilibiliH5Home.jpeg)  

类似于这种由固定外框+tab切换内部滚动div的结构，在丢进嵌套下拉刷新的WebView内而不作任何处理的话，无论内部div滚动到头与否，上拉必然触发下拉刷新。
因为外框内容的高度是固定的，正好适配到WebView的高度，而WebView也没有对这种内部滚动进行监测，这就导致Android大部分下拉刷新控件使用的```View.canScrollVertically(-1)```方法永远返回false。
这种情况下，当上拉的时候，WebView判断内容已经到头，就会让套在外边的RefreshLayout认为可以触发下拉刷新了。  
不过需要指出的一点是，虽然我们这种简易的RefreshLayout套WebView布局在这种Web页面上会有滑动冲突，但同样的页面放到诸如Chrome之类的系统浏览器app里却没有任何问题。这说明肯定是有什么办法能够处理这个冲突的。

就这个问题我在网上查了查，中文网络上的内容实在是乏善可陈，唯一一篇比较有价值的，是说跟前端那边商定一个滚动布局的id，然后native通过js注入，去查对应id的div是否滚动到顶部。
```java
    // inner_scroll_box是web页面内部滚动布局的id
    getViewBinding().webView.setOnTouchListener((v, event) -> {
        getViewBinding().webView.evaluateJavascript("document.getElementById(\"inner_scroll_box\").scrollTop", new ValueCallback<String>() {
            @Override
            public void onReceiveValue(String value) {
                Log.d("webview", "scrollTop: " + );
            }
        });
        return false;
    });

```
这个方法我没有考虑过落实，普适性太差，Web那边重构一下，或者使用的是第三方的页面，都可能导致问题。从另一点来说，没有滑动冲突的系统浏览器app，他们要适配的页面千千万，很明显不会是采用这种方法来解决问题的。

继续检索，so的老哥提出了一种解决方法，检查WebView的overscroll状态。就是对RefreshLayout套WebView这种布局做特殊处理，触控事件先全交给WebView，等上滑到WebView overscroll触发了，就判定RefreshLayout可以启用了，下拉刷新可以触发了。这个OverScroll状态，在WebView的```onOverScrolled()```方法中执行
```java

public class CustomWebView extends WebView {

    @Override
    protected void onOverScrolled(int scrollX, int scrollY, boolean clampedX, boolean clampedY) {
        super.onOverScrolled(scrollX, scrollY, clampedX, clampedY);
        if( clampedX || clampedY ){
            //Content is not scrolling
            //Enable SwipeRefreshLayout
            ViewParent parent = this.getParent();
            if ( parent instanceof PullRefreshLayout) {
                ((PullRefreshLayout)parent).setEnabled(true);
            }
        }
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        if(event.getActionMasked()==MotionEvent.ACTION_DOWN){
            //Disable SwipeRefreshLayout
            ViewParent parent = this.getParent();
            if (parent instanceof PullRefreshLayout) {
                ((PullRefreshLayout)parent).setEnabled(false);
            }
        }
        return super.onTouchEvent(event);
    }

}

```

这个方法让内部滚动div滚动到头之后才能触发下拉刷新。但并没有完全解决整个滑动冲突的体验问题。使用中会出现滑动到头之后，继续下拉，下拉刷新无响应，下拉很多次才会触发下拉刷新。
这个问题是由于当对下拉刷新布局开关的控制放进```onTouchEvent()```这样高频触发的方法内部后，每一次手指点击屏幕就把下拉刷新给关了。要手指继续往下划，才会触发overscroll判断，开启下拉刷新，但此时一个完整的touch动作链条（down-move-up）已经被消耗了，自然不会触发下拉刷新布局响应。只有多试几次赌脸触发。  
很明显这个体验是很垃圾的，还可以优化。既然明白原理，那么优化的点自然也很好找到：修改```onTouchEvent()```中refreshLayout开关的判断逻辑。
```java

public class CustomWebView extends WebView {

    private float initY;
    private boolean overScrolled = false;
    private boolean refreshEnabled = false;

    @Override
    protected void onOverScrolled(int scrollX, int scrollY, boolean clampedX, boolean clampedY) {
        super.onOverScrolled(scrollX, scrollY, clampedX, clampedY);
        if( clampedX || clampedY ){
            overScrolled = true;
        }
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        ViewParent parent = this.getParent();
        if (parent instanceof PullRefreshLayout) {
            if(event.getActionMasked() == MotionEvent.ACTION_DOWN) {
                initY = event.getY();
            }
            else if(event.getActionMasked() == MotionEvent.ACTION_MOVE) {
                float dy = event.getY() - initY;
                // 当手指往上划时直接认为下拉刷新可以关闭
                if (dy < 0) {
                    overScrolled = false;
                    refreshEnabled = false;
                    ((PullRefreshLayout)parent).setEnabled(false);
                }
                // overscroll + 下拉刷新未开启，达到开启下拉刷新条件
                if (overScrolled && !refreshEnabled) {
                    refreshEnabled = true;
                    ((PullRefreshLayout)parent).setEnabled(true);
                    // 根据当前位置创造一个ACTION_DOWN事件发送给parent，让其能够处理一个完整的touch链执行下拉刷新
                    MotionEvent obtain = MotionEvent.obtain(event);
                    obtain.setAction(MotionEvent.ACTION_DOWN);
                    dispatchTouchEvent(obtain);
                    ((PullRefreshLayout)parent).dispatchTouchEvent(obtain);
                }
            }
        }
        return super.onTouchEvent(event);
    }

}

```

经过这个修改优化之后，基本可以达到与浏览器app一致的下拉刷新体验，且不需要对页面单独适配。  
缺点是需要继承WebView并且跟下拉刷新布局耦合，当使用自定义下拉刷新布局的时候还需添加额外的判断逻辑，如果遇到难以进行继承操作的第三方自定义WebView会比较麻烦。  
不过暂时我还没想出更好的办法来处理这个，有更优解的朋友欢迎在评论中点出。