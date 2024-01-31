---
layout:     post
title:      使用FileWatcher实现flutter代码自动生成
subtitle:   
date:       2023-11-28
author:     lx8421bcd
header-img: img/post-bg-android.jpg
catalog: true
tags:
    - Android
---
## 前言

之前在搭建Flutter项目框架一文中提过一个观点 "资源引用化"，即像Android、iOS等原生开发一样，将图片、多语言适配文本等资源生成一份Path-Constants映射表，然后在项目中以使用常量引用的方式获取这些资源的路径/值。这样一旦资源路径或命名发生变更，只需调整常量对应路径的值，无需变更业务代码。

但这就有一个问题，「修改常量对应路径值」的这个操作还是手动的，手动就不可避免的会出现人为差错，哪个老哥代码写麻了忘记改，字符串又不会报错，也发现不了，必须得想个办法让「生成映射表」这一过程自动化。

这时候就用到了[flutter_gen](https://pub.dev/packages/flutter_gen)和[flutter_intl](https://pub.dev/packages/intl)。只需要运行一下 `build_runner`就可以自动生成一套，与Native开发体验快差不多了。还剩一个问题，Terminal执行 `build_runner`命令可否自动化？

思路其实挺简单：监听项目下的资源目录，如果发现目录内有文件变更、增减，就触发一个事件，执行一下 `build_runner`命令，生成新的映射文件。下一步的解题点就很明确了，IDE中有没有插件能支持监听某一目录/文件的变化并执行指令？

很幸运，有，我很快就找到了FileWatcher，而且FileWatcher在Android Studio和VSCode中均有较好的支持。

## 配置

本文主要讲解一下Android Studio下的配置过程。    

首先项目的状态应达到集成完毕flutter_gen/flutter_intl，运行 `build_runner`即可以生成映射文件的状态 。   
在Android Studio  Settings → Plugins → MarketPlace中下载并安装 `File Watchers`。    
配置自动运行 `build_runner`的脚本，mac下.sh，windows下就.bat，里面就放build命令，脚本文件建议存放在项目根目录下    
```shell
# 如果你的脚本需要在项目的某一路径下执行，最好先跳转到当前项目的根目录
  cd $(dirname "0") || exit    

  dart run build_runner build
```

对于flutter_intl，如果你是使用flutter_intl插件来做生成映射表的话，可以使用如下命令编写脚本
``shell
  cd $(dirname "0") || exit    

  flutter pub global run intl_utils:generate
```

一般来说建议一个类型资源编写一个脚本，不要把build命令全部放一起，不然你改一下文案，图片映射表跟着一起重新build一遍，蛋疼不……    
之后在Settings → Tools → File Watchers下配置监听    
![File Watcher项目配置](https://raw.githubusercontent.com/lx8421bcd/lx8421bcd.github.io/master/img/file_watcher/file_watcher_config_example.png)      
简单叙述一下本图中的配置项：
* Name - 项目名字，没啥好说的
* Files to Watch
  * File type - 文件类型，一般来说选Any就行了
  * Scope - 监听文件路径配置，选择旁边的`...`会打开如下弹窗，根据你当前的项目路路径选择include/exclude就行了，IDE会帮你生成对应的pattern
    ![File Watcher Scope配置](https://raw.githubusercontent.com/lx8421bcd/lx8421bcd.github.io/master/img/file_watcher/file_watcher_scope_example.png) 
    注意勾选"Share through VCS"，这样Android Studio会在你当前项目的`.idea/scopes/`路径下生成监听路径的配置文件，可用于多端，多人协作时的同步。
* Tool to Run on Changes - 这个就是选择在监测到监听目录的文件发生变化时所需执行的操作，在本项目的需求中，我们选择执行对应的脚本即可。
  以上图为例，在Program下输入脚本路径，其中`$ProjectFileDir$`即是当前项目的根目录。

完成上述配置并保存，一个FileWather监听就配置好了，理论上来说已经可以在监听路径下修改文件测试了。    

注意，对于Mac或Linux系统，有可能存在脚本没有权限运行的情况，需要修改一下脚本文件的权限    
```shell
  # 这里以脚本存放在项目根目录为例
  cd $(dirname "0") || exit    

  chmod 777 l10n_build.sh

  chmod 777 flutter_gen_build.sh
```

完活，修改文件测试

## 配置同步

对于File Watcher配置，我们当然不希望项目组每个人都整一回，或者每换一次电脑都配置一回，如果有办法能在新的开发环境下导入配置自然是最好。
所幸，File Watcher提供了这种支持，操作并不复杂，这里简要说一下步骤

首先在项目中配置gitignore, 如果你的项目将`.idea`路径排除了的话，需要保留一下上文提到的scope配置存储的路径

  ```gitignore
  .idea/
  !.idea/scopes/
  ```

然后将当前项目中的File Watcher配置导出
![File Watcher 导入导出](https://raw.githubusercontent.com/lx8421bcd/lx8421bcd.github.io/master/img/file_watcher/import_export.png) 
导出之后应该会生成一份`file_watchers.xml`，建议将这个文件存放在项目根目录下。    
git提交所有相关配置，包含脚本、`file_watchers.xml`配置，`.idea/scopes/`的监听路径配置等。    
在新的工作环境导入File Watcher配置，由于scope下监听配置文件都是现成的，直接在File Watcher插件中导入`file_watchers.xml`就可以了。    
脚本权限，与配置一样，如果运行脚本时出现权限问题，参考上面的命令给脚本文件提权。    

其实不止是资源文件，诸如Json序列化/反序列化之类的需要自动生成代码的需求，都可以使用File Watcher监听来完成，自行配置即可