---
layout:     post
title:      关于Code Review
subtitle:   读“谷歌工程实践”review部分有感
date:       2020-05-13
author:     lx8421bcd
header-img: img/post-bg-unix-linux.jpg
catalog: true
tags:
    - 技术随笔
---

#### 读后感

偶然间看到前些日子谷歌开放的工程实践文档，目前里面只有关于code review相关的内容，虽说没有诸如“如何搭建CR系统怎样部署自动化CR”之类的干货，只是对code review这个制度如何进行，如何进行的更顺利的一些经验之谈，但研读一番下来发现，其实无论做不做code reivew都是有一定收获的。  
这一系列文章其实强调了关于CR的几个“要点”，或者说“精神”：  

* 细粒度提交
* CR应尽量宽容，但不应为了宽容而降低标准
* 不要把CR没过当作是被针对，不要因此产生心理负担
* 不要拖延



**细粒度提交**应该是这一系列实践经验中最重要的一个原则了，文章从审查者和开发者两方面花了大量的篇幅叙述这一点的重要性，总的来说，这个原则就是

    每一次CL（change list）在保持可完整运行的前提下，应该尽可能的小

个人认为这是一个非常重要的编程习惯，而不仅仅是为了CR而提出的要求，工作多年遇到的同事很少有保持这一习惯的，除非有合作需要同步代码，不然都喜欢1天提交一次，甚至一个需求（5～10天的开发周期）就提交2～3次，一次commit十几个文件修改。不出问题还好，遇到问题要回滚，或者后期有什么bug要追溯提交记录就头疼不已。  
细粒度提交出了麻烦一点，剩下的全是好处，最主要的一点就是将单一改动对其它改动的影响降至最低，基于这点又带来了其它好处：

* commit 描述好写，1～2个文件的改动，只需要简要介绍一下就能让人明白
* 合并方便，高频次小改动提交产生冲突的可能性更小，处理冲突也更容易
* 回溯容易，基于上面两条，如果代码出现异常，从一次次小CL里找问题难度远低于在大CL里面慢慢翻
* 更难产生bug，小CL更容易测试，也更容易审查，产生bug的几率也更小

Google工程实践上还列出了其它优点，这里不一一赘述。我认为，细粒度提交（小CL）是一个无论工作内有CR制度与否都要坚持的编程习惯，长期坚持对提升自己的代码质量非常有帮助，也是工作时的省力之道。  



**CR应尽量宽容**，这一点我认为是一个部门在试图实施CR的时候一定要让审查者明白的一个道理，换句话说，CR针对的点主要是原则性问题，比如code style是否符合规范，是否存在错误设计，是否存在内存泄漏等隐患之类；除此之外，CR应尽量宽容，不要把个人的设计风格当作代码审查标准，也不要在审查时过于严苛。  

CR追求的是“持续改进”，而非“一步到位”  

没有人能一次写出完美的代码，哪怕现在看上去完美，将来随着版本迭代上来也会逐渐产生问题，过于严苛的审查很可能会降低开发效率，并导致过度设计。  

但也不能因此走向反面，即为了提升速度而降低审查标准，这样可能会导致代码审查流于形式，无论项目再紧急原则问题必须坚持，这一点可以与“不要拖延”联系起来说，即尽量避免项目中出现破窗。    

对于大部分业务开发人员来说，需求是不断的，一个不好的实现写下去，除非这个需求相关的代码变更，否则很难再腾出时间去修改、优化，每一次开特例都是在为今后的修改埋坑。比如很典型的一个“don't repeat your self”。一套业务本来已经有实现代码，又有另外的地方要用，按理来说，应该将其封装成工具提供对外接口供多处使用以方便需求变更。但开发人员为了省事，直接复制了一份应付新需求；这样的实现可以通过吗？通过了就意味着，如果这个实现有bug，复制了几处就会有几处的业务受到影响；如果需求有变更，复制了多少就要改多少……

所以，特例，绝对不能开，短期来说，这增加了开发时间，但长期来看，这是在节约时间。



其它几方面主要是从开发人员（审查者，开发者）的心态上来说的，我也就没什么感想了，Google在照顾开发者心态方面真的用了很大的工夫，保证开发者的工作积极性真的很重要啊哈哈。



#### 我所经历的CR

一边看一边思考，就让我想起我司的code review经历，那时候我还在游戏部门，技术是部门内的主导力量，领导也是技术出身，对于代码质量这块是比较重视。但游戏程序员终归是比较忙的，我们既要负责移动端和PC端的新游戏开发，又要负责维护已经上线的移动/PC游戏，忙的时候在开发一款游戏的同时又要负责3～4款游戏的维护，那真的是就差睡在公司了，我组的技术老大就比我惨点，他还负责一个新游戏平台开发，连着3个月无休。在这种情况下想坚持每次CL都有CR，基本是强人所难了。

但我部门还是坚持去做CR，流程是这样的，抽一个大家都不忙的时间，比如周五下午，选定一位组员近期所开发的一个模块，以开会的形式向研发组讲解代码设计思路，其他组员一边听一边看一边提意见讨论，最后将修改意见整理修改，再由组长验收。当然了，这样的CR会议只挑淡季做，忙的季节是没时间整的……

我个人认为这是一种极其失败的CR形式，这种CR形式基本违背了Google工程实践中所有的要点。如果研发团队的CR只能做到这种，那还不如不做。当然我不是奉Google实践为圣经，我就说说我个人看到的问题。

首先这种对整个模块进行review的形式就完全违反了CR里面的细粒度提交原则，一整个模块可能包含几个甚至十几个上千行的源文件代码，一下子甩这么一大个玩意到审核员头上让他用几十分钟时间过一遍并在CR会议上提出见解是一件非常难的事，审核者对模块的理解高度依赖于开发者的讲述，如果开发者口才不太好讲不清楚，那没有参与此需求开发审核员基本无法在这一下午的时间内理清模块逻辑，接口设计，最后大家就只能就一些无关痛痒的问题提出点意见。

事实上我参与的需求会最后基本都会走到这一步，大家最后都在花时间讨论这个ifelse是不是应该写成else - if …… 当然我不是说这样的讨论没有意义，只是，这本是一个根据code style就能决定的问题，值得花全组人的时间来争论么…… 相反，更值得去研讨的诸如是否应该使用多线程/协程，模块间耦合等影响更大的设计却因为审查者难以短时间内了解模块逻辑和需求而无法审查。

一次CR一个模块的另一缺点就是会将CR过程拉的极长，模块比较小，没什么大问题倒还好，1个小时左右就讲完了，如果说开发者讲解的不太清楚，大家对着由异议的地方讨论一下，那就是3小时起步了，从午休完之后直接开到下班吃完饭也不是不可以。请各位自己想象，你开3个小时以上的会是什么感受，开到最后你还能提什么建设性意见么？但你又不能表现的不耐烦，最后的结果就是又开始讨论起if-else了……

对比一下上文工程实践的要点，这样的CR形式缺点在哪？

1. 拖延，CR时代码已入库，有的甚至以上线，加大了改动风险
2. 粗粒度，CR耗时巨长，审核员无法在短时间内理清模块逻辑，导致难以提出建设性意见。
3. 由于2的存在，这种CR只能避免基本原则性的问题（拼写、格式、code style）可以认为是宽容过度了

那么这种形式的技术会是不是一无是处呢？其实也不是，我认为这种技术会在业务无关的基础组件上相当有价值，与会成员限制在同一业务组内，组件开发者讲解实现方案其实就是在介绍使用方法和实现思路，而其它组员在学习如何使用组件时也可以帮开发者捉虫。

严格来说这种会议并不能算CR，而应该算技术方案会议。基于上文提到的缺陷，我认为这种技术会议更应该开在编码前，丢开if-else级别的细节，大致统一开发方向以及实现方案，为之后工作中的CR确定标准。



#### 如何推进CR

后来到了直播部门，当然都是一个公司下的部门，作风也类似，也玩上面的“代码评审会”那一套，只不过频率要低很多了，差不多半年一次…… 

由于后面发生过几次因为组员写出低级错误没有导致线上崩溃的事故，技术老大希望我们这边能推进一下CR制度，具体落实最后交给我来实施。做Android不得不说，对比cocos2d-lua的辣鸡IDE，Android Studio真的是让我回到了现代。

Android Studio本身的lint工具提供了非常强大的代码质量检查，小到拼写错误，大到潜在的空指针和循环引用都能做出有效的警告，按照 [通过 lint 检查改进代码](https://developer.android.com/studio/write/lint) 来配置lint检查，并通过报错等手段强制让开发者将低级的原则性错误修复。

这也是我认为一个部门要推进CR的第一步，通过自动化手段将原则性问题先拦下来。这样做，第一是能够让审查者专注于审查更深层更有价值的内容，比如代码逻辑、接口设计。第二是通过制度保证了提交到代码库代码的质量底线，哪怕再忙，这条底线也是得遵守的，因为修改CR配置本身也是需要开发成本的，遵守并养成习惯是最佳选择。

之后关于如何审查的问题，我想Google工程实践比我写的详细就没必要复读了。

在推进CR的过程中我发现一个很大的问题就是一个开发组想要推进CR，除了CR制度之外很重要的一点是开发人员的水平起码得在良好以上，如果是刚刚及格，那么审核员就会很头疼，长期下来对审核员的精力摧残程度十分可观。比如让一位老哥写socket通信的管理工具，他写了一天之后提交给我看的是一个单纯的连接上socket，读字节流并转成Java数据对象的玩意……

在我反复跟他沟通了一整天之后，他终于提交给我了一个还算像样的工具类。但说实话，这劳累程度跟我亲自写一个没啥区别了，甚至更累一些……

后来随着直播平台没落，外加不断尝试其它路子导致的不断做新项目，项目组疲于奔命，我这已经建立起来的CR制度也逐渐崩坏，尝试CR伸出去的慢慢脚又收了回来，最终还是没有迈出这一步，比较可惜。



#### 写在最后

就我个人的经历来说，Code Review大部分公司都是有的，但能够形成制度并长期执行的team真的是少之又少。特别是在国内伪敏捷横行，整个研发链条很多时候从源头上就有问题的情况下，高质量代码，特别是业务代码的价值被降低了，不合理的项目管理和工作流反向增加了CR制度推行的难度。

CR制度更多的是针对一个研发团队的代码质量控制体系，而对于开发者个人来说，我觉得自律是比你所在部门有没有CR更重要的东西，保持良好的编程习惯并尽可能的在深思熟虑后编码，即使没有CR制度，本身也可以维持一定的代码质量；相反，哪怕有CR制度，如果开发者只在不触犯基本原则的情况下放飞自我，对CR制度来说也是一种破坏。

