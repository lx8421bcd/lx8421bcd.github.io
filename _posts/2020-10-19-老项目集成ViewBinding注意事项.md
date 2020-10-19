---
layout:     post
title:      老项目集成ViewBinding注意事项
subtitle:   
date:       2020-10-19
author:     lx8421bcd
header-img: img/post-bg-android.jpg
catalog: true
tags:
    - Android
---

ViewBinding在2019年的谷歌开发者大会上推出并在Android Studio 3.6之后提供支持。在这之前我们处理XML布局到Java代码的映射要么使用最古老的findViewById要么使用ButterKnife。  

findViewById()作为AndroidSDK最原始的手法自然是不会有什么大问题的，除了写着麻烦，以及潜在的类型转换隐患。ButterKnife其实也并没有比findViewByid()方便到哪里去，除非使用了如ButterKnifeZeleny这样自动生成Java映射的插件。另一方面，ButterKnife在library module中使用存在问题，必须使用```R2.id...```这样的形式指向对应的XML布局ID，在做代码追踪方面有一定的麻烦。  

ViewBinding彻底的解决了这两个问题，因为映射工作就像R文件生成一样交由IDE自动生成，不存在类型转换的安全问题，也能尽量将XML-Java对象这块的异常问题暴露在编译期。所以，对于目前仍然在维护的Java Android项目来说，无论是从提升项目可维护性的角度，还是从开发者偷懒的角度，我都建议尽快集成ViewBinding，并跟随项目渐进迭代替换掉ButterKnife。

集成方式以及使用注意事项 [谷歌已经说的很明白了](https://developer.android.com/topic/libraries/view-binding?hl=zh-cn)， 这里不再赘述。

这一次我就是借老项目重构集成ViewBinding，不过这个项目老的有点可以，已经维护了6～7年了。简要说说我在集成中遇到的问题：

**XML携带BOM信息的问题**  
有些老到是从eclipse迁移过来的项目，XML文件中可能会带有BOM信息，集成ViewBinding后在build的过程中会报NullPointerException，基本没有多的异常信息，我也是查了好久才知道。
这种情况使用工具将BOM信息擦除即可  
Windows下自行搜索“去除BOM小工具”对res各个子目录下的xml文件进行擦除即可。  
mac下可以使用shell的相关命令进行擦除，参考：[OSX使用sed移除UTF-8中的BOM](https://aillieo.cn/post/2018-07-02-osx-sed-remove-bom/)  

**Replugin**  
对于集成了Replugin的项目，gradle版本最多升到3.6.x，4.0以后Gradle内部的函数变更会导致Replugin在build期间失败。  
> A problem occurred configuring project ':app'.
>
> groovy.lang.MissingPropertyException: No such property: scope for class: com.android.build.gradle.internal.variant.ApplicationVariantData
目前无解，只能等之后看看Replugin会不会有更新适配。  

**无用的layout文件**  
老项目经过多年迭代会留下一些不用的layout文件，其中引用到的自定义控件可能已经变更路径或被删了，这种文件以前Build是不会出错的，但是集成了ViewBinding之后，Build到这些文件时会产生异常，即无法找到对应的View进行映射。  
如果layout没有用了建议直接删掉，或者在根标签添加ignore属性让ViewBinding忽略此文件。  

