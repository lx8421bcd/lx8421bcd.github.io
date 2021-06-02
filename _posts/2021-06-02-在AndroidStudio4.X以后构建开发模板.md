---
layout:     post
title:      在AndroidStudio4.X以后构建开发模板
subtitle:   
date:       2021-06-02
author:     lx8421bcd
header-img: img/post-bg-android.jpg
catalog: true
tags:
    - Android
---

### 前言
长期以来Android Studio除了官方模板之外并没有提供官方的、比较完善的模板编辑系统，而官方模板则是基于FreeMarker。如果用户有自定义模板需求，则基本上是基于官方的模板修改，然后按分类与Android Studio原生的模板放在同一路径下。  
这种玩法优点是比较简单，大框架摆在那里，个性化的需求小修小改就可以了，缺点是没什么好用的编辑工具，只有文本编辑器，关键字提示是不存在的。另外一个问题是每次Android Studio版本更新过后，这些非官方的东西会在安装过程中被移除，要重新导入。  
之前为了应付公司的新模块开发我也整了一套基于上述操作的模板，还写了shell/bat脚本用于在更新后一键导入，但更新了Android Studio 4.2之后，一键导入脚本全部报错，说找不到路径。我一看原来的tempaltes路径全没了，再一查，好么，原来的模板实现模式已经改了。  

从Android Studio 4.1开始谷歌采用Geminio，使用Intellij Plugin的形式用kotlin编写模板，老的FreeMarker模板目前是无法在官方支持的体制下使用了。虽然说操作起来比FreeMarker麻烦一点，但起码是有IDE提示知道能用哪些功能了……

目前网上有一些继续使用FreeMarker的方法，但鉴于本文主要讲新的模板编辑方法，这里就不赘述了。

### 创建项目
首先从Github上把 [Intellij Platform Plugin Template](https://github.com/JetBrains/intellij-platform-plugin-template) 项目fork下来。这算是一个最简的插件项目工程模板了。  
然后就是修改项目名称和相关配置属性：
* 修改位于```settings.gradle.kts```里的```rootProject.name=name```为自己模板项目的名称
* 修改```gradle.properties```里的项目配置
    ```
    # 插件制作者/所属组织
    pluginGroup = lx8421bcd
    # 插件名称
    pluginName = quickdevtemplates
    # 插件版本
    pluginVersion = 1.0.0
    ```
* 修改项目包名， 将```src/main```里的代码包名从```org.jetbrains.plugin.template```改成你想要的包名（比如我为QuickDevFramework改为```com.lx8421bcd.qdftemplate```）。然后修改```src/main/resources/META-INF/plugin.xml```文件的配置
    ```xml
    <idea-plugin>
        <id>com.lx8421bcd.qdftemplates</id>
        <name>QDFTemplate</name>
        <vendor>lx8421bcd</vendor>
    
        ......
    
    </idea-plugin>
    ```

### 导入依赖库
1. 从Android Studio安装目录中把```wizard_template.jar```复制出来，如果用的是mac，以下命令供参考：
    ```shell
    cp /Applications/Android\ Studio.app/Contents/plugins/android/lib/wizard-template.jar ~/Desktop
    ```
2. 在插件项目根目录中创建一个```lib```文件夹，把```wizard_template.jar```放进去。
3. 编辑项目的```build.gradle.kts```配置文件，加入依赖
    ```gradle
    dependencies {
        detektPlugins("io.gitlab.arturbosch.detekt:detekt-formatting:1.17.1")
        // 添加依赖库
        compileOnly(files("lib/wizard-template.jar"))
    }
    ```
4. 在```plugin.xml```中添加依赖项
    ```xml
    <idea-plugin>
        ......
    
        <!-- Product and plugin compatibility requirements -->
        <!-- https://plugins.jetbrains.com/docs/intellij/plugin-compatibility.html -->
        <depends>com.intellij.modules.platform</depends>
        <depends>org.jetbrains.android</depends>
        <depends>org.jetbrains.kotlin</depends>
    
    </idea-plugin>
    ```
5. sync project

### 修改插件项目功能入口
1. 编辑```MyProjectManagerListener.kt```，配置插件在项目初始化时的操作
    ```kotlin
    
    internal class MyProjectManagerListener : ProjectManagerListener {
    
        override fun projectOpened(project: Project) {
            projectInstance = project
            project.getService(MyProjectService::class.java)
        }
    
        override fun projectClosing(project: Project) {
            projectInstance = null
            super.projectClosing(project)
        }
    
        companion object {
            var projectInstance: Project? = null
        }
    }
    
    ```
2. 在与```listener```同级的package下创建一个```other```package
    ```
    --- com.lx8421bcd.qdftemplate  
     |--- listener
     |--- services
     |--- other
    
    ```
3. 创建TemplateProvider继承实现，用于让IDE查找本插件项目提供的模板
    ```kotlin
    
    package other
    
    import com.android.tools.idea.wizard.template.Template
    import com.android.tools.idea.wizard.template.WizardTemplateProvider
    import other.activity.SimpleViewBindingActivityTemplate
    import other.fragment.SimpleViewBindingFragmentTemplate
    
    class QDFPluginTemplateProviderImpl : WizardTemplateProvider() {
    
        override fun getTemplates(): List<Template> = listOf(
            // 在项目中创建的模板需要在这个列表内声明
            SimpleViewBindingActivityTemplate,
            SimpleViewBindingFragmentTemplate,
        )
    }
    
    ```
4. 修改```plugin.xml```，添加对TemplateProviderImpl的声明，完整```plugin.xml```如下:
    ```xml
    <idea-plugin>
        <id>com.lx8421bcd.qdftemplates</id>
        <name>QDFTemplate</name>
        <vendor>lx8421bcd</vendor>
    
        <!-- Product and plugin compatibility requirements -->
        <!-- https://plugins.jetbrains.com/docs/intellij/plugin-compatibility.html -->
        <depends>org.jetbrains.android</depends>
        <depends>org.jetbrains.kotlin</depends>
        <depends>com.intellij.modules.platform</depends>
    
        <extensions defaultExtensionNs="com.intellij">
            <applicationService serviceImplementation="com.lx8421bcd.qdftemplates.services.MyApplicationService"/>
            <projectService serviceImplementation="com.lx8421bcd.qdftemplates.services.MyProjectService"/>
        </extensions>
    
        <applicationListeners>
            <listener class="com.lx8421bcd.qdftemplates.listeners.MyProjectManagerListener"
                    topic="com.intellij.openapi.project.ProjectManagerListener"/>
        </applicationListeners>
    
        <extensions defaultExtensionNs="com.android.tools.idea.wizard.template">
            <wizardTemplateProvider implementation="other.QDFPluginTemplateProviderImpl" />
        </extensions>
    
    </idea-plugin>
    ```

至此一个模板插件项目基本配置完成，之后编写模板只需将模板构建方法添加到TemplateProvider实现类的列表中即可。

### 编写模板
以下用Activity模板为例：
```kotlin
package other.activity

import com.android.tools.idea.wizard.template.*
import com.android.tools.idea.wizard.template.impl.activities.common.MIN_API
import com.android.tools.idea.wizard.template.impl.activities.common.generateManifest
import com.intellij.util.xml.DomManager

//用于提供默认PackageName
val defaultPackageNameParameter
    get() = stringParameter {
        name = "Package name"
        visible = { !isNewModule }
        default = "com.lx8421bcd.example"
        constraints = listOf(Constraint.PACKAGE)
        suggest = { packageName }
    }
// Activity模板Builder
val SimpleViewBindingActivityTemplate
    get() = template {
        revision = 1
        name = "Simple ViewBinding Activity"
        description = "基于ViewBinding基类的Activity模板"
        minApi = MIN_API
        minBuildApi = MIN_API

        category = Category.Other
        formFactor = FormFactor.Mobile
        screens = listOf(WizardUiContext.ActivityGallery,
            WizardUiContext.MenuEntry,
            WizardUiContext.NewProject,
            WizardUiContext.NewModule
        )

        lateinit var layoutName: StringParameter

        val activityClass = stringParameter {
            name = "Activity Name(不包含\"Activity\")"
            default = "Main"
            help = "只输入名字，不要包含Activity"
            constraints = listOf(Constraint.NONEMPTY)
        }

        layoutName = stringParameter {
            name = "Layout Name"
            default = "activity_main"
            help = "请输入布局的名字"
            constraints = listOf(Constraint.LAYOUT, Constraint.UNIQUE, Constraint.NONEMPTY)
            suggest = { activityToLayout(activityClass.value.toLowerCase()) }
        }

        val packageName = defaultPackageNameParameter

        widgets(
            TextFieldWidget(activityClass),
            TextFieldWidget(layoutName),
            PackageNameWidget(packageName)
        )

        recipe = { data: TemplateData ->
            simpleViewBindingActivityRecipe(
                data as ModuleTemplateData,
                activityClass.value,
                layoutName.value,
                packageName.value)
        }
    }

// 用于向项目写入模板文件的方法
fun RecipeExecutor.simpleViewBindingActivityRecipe(
    moduleData: ModuleTemplateData,
    activityClass: String,
    layoutName: String,
    packageName: String
) {
    val (projectData, srcOut, resOut) = moduleData
    val ktOrJavaExt = projectData.language.extension
    // 插入manifest声明
    generateManifest(
        moduleData = moduleData,
        activityClass = "${activityClass}Activity",
        activityTitle = activityClass,
        packageName = packageName,
        isLauncher = false,
        hasNoActionBar = false,
        generateActivityTitle = false,
    )
    // 生成activity文件
    val activityFile = simpleViewBindingActivityKt(projectData.applicationPackage, activityClass, packageName)
    save(activityFile, srcOut.resolve("${activityClass}Activity.${ktOrJavaExt}"))
    // 生成xml布局文件
    val xmlFile = simpleViewBindingActivityXml(packageName, activityClass)
    save(xmlFile, resOut.resolve("layout/${layoutName}.xml"))
}

/*-------------------- activity code generate function ----------------------*/
fun simpleViewBindingActivityKt(
    applicationPackage:String?,
    activityClass:String,
    packageName:String
)="""
package $packageName
import android.os.Bundle
import com.linxiao.framework.architecture.SimpleViewBindingActivity
import ${applicationPackage}.R
import ${applicationPackage}.databinding.Activity${activityClass}Binding
class ${activityClass}Activity : SimpleViewBindingActivity<Activity${activityClass}Binding>() {

     override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        initView()
        
    }

    private fun initView() {

    }
} 
"""

/*-------------------- layout xml code generate function ----------------------*/

fun simpleViewBindingActivityXml(
    packageName: String,
    activityClass: String
) = """
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout 
xmlns:android="http://schemas.android.com/apk/res/android"
xmlns:app="http://schemas.android.com/apk/res-auto"
xmlns:tools="http://schemas.android.com/tools"
android:layout_width="match_parent"
android:layout_height="match_parent"
tools:context="${packageName}.${activityClass}Activity">
    
    
</RelativeLayout>
"""

```

### 导出插件、安装
在Android Studio（Intellij ieda亦可）工具栏的Run上选择“Run Plugin”，build完成后会在项目的```build/libs/```目录下生成插件Jar包。

安装插件：
1. 打开Android Studio设置
2. 在Plugins一栏点击齿轮图标，选择“Install Plugin from Disk”
3. 选择生成的Jar包安装
4. 重启Android Studio，完成安装

### 小结
本来自定义开发模板的本质也就是根据模板指引面板的输入，将一堆字符串改改保存到文件中存到指定位置，用不上太复杂的功能。  由于是使用```wizard_template.jar```包内提供的API直接编写kotlin代码，加之有IDE环境下，开发起来还是要方便不少，最起码不会在文本编辑器上抓瞎编写。其实这个模板不仅限于生成Activity、Fragment这些Android组件，也可以用于配置更复杂模板，例如MVVM架构下配套的ViewModel、带有列表的页面等等，只要参照生成文件的方法写好，保存到对应路径就行。

需要注意的一点是，由于编辑Android组件模板内容其实就是编辑字符串，所以错误提醒是不存在的。咱们用模板就是图个一步到位，模板生成代码一片红看着都烦死。外加现在每次更新模板插件都需要重新打包，重新安装，重启IDE，非常麻烦，所以建议在编写复杂模板的同时最好开着一个Android项目用于测试模板代码，在项目环境下没问题了，再复制到模板中去。

另外一点是，不建议搞巨无霸模板，一般来说，一套模板插件针对一个项目，或者使用相同基础库逻辑类似的项目（比如一个项目组所负责的多个项目）。还是那句话，谁都不喜欢模板生成个代码一片红，如果要针对差异化很大的多个项目开发通用模板，那就有更多内容需要靠模板引导页面灵活配置，如果最后用模板生成个页面整的像填报表一样的，那估计这模板也没谁想用吧……
