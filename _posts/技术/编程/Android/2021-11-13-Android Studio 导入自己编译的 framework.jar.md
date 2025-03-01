---
layout: post
title: "Android Studio 导入自己编译的 framework jar"
date:  2021-11-13 18:58:12 +0800
categories: ["技术", "编程", "Android"]
tag: ["Android", "framework.jar"]
---

网上的文章大多是 Android Studio 2.x 环境，实行起来，坑比较多。
本文适用于 Android Studio 3.x 及以上，亲测可行。


## 一、编译生成 framework.jar 包
系统级 App 开发，很多时候需要访问 framework 层隐藏的接口（接口前的注释里加了@hide）,有时候甚至是定制的系统，在 framework 层加入了新的接口，为了使用这些接口，需要自己编译 framework 的源码生成 jar 包，如果编译 debug 版本，直接把 `out/target/product/projectXX/system/framewor framework.jar` 下面拷出来，
如果是 user 版本:
Android N/O:
`out/target/common/obj/JAVA_LIBRARIES/framework_intermediates/classes.jar`
Android P/Q:
`out/soong/.intermediates/frameworks/base/framework/android_common/combined/framework.jar`
Android R:
`out/soong/.intermediates/frameworks/base/framework-minus-apex/android_common/combined/framework-minus-apex.jar`

## 二、将 framework.jar 导入 Android Studio 工程

1. 将framework.jar放在Module的libs下面<br/>

![add lib](/assets/images/技术/编程/Android/Android%20Studio%20导入自己的%20framework.jar/framework_jar.png)

2. 添加对 framework.jar 的依赖
在 module 的 build.gradle 文件中添加：

```
repositories {
    flatDir {
        dirs 'libs'
    }
}
```

同时， dependencies 中添加依赖：

```
dependencies {
    compileOnly fileTree(dir: 'libs', include: ['*.jar'])
}
```

**注意：**
使用 compileOnly（默认是 implementation），compileOnly 表示 jar 包只参与编译，不会打包进去。

如果同时还需要依赖其它 jar 包，并且其它需要打包进去，则分开对每个 jar 包单独进行依赖。

```
compileOnly files('libs/framework.jar')
implementation files('libs/XXX.jar')
```

3. 在 module 的 build.gradle 文件中添加如下代码：

```
gradle.projectsEvaluated {
    tasks.withType(JavaCompile) {
        Set<File> fileSet = options.bootstrapClasspath.getFiles()
        List<File> newFileList =  new ArrayList<>();
        //"../framework.jar" 为相对位置，需要参照着修改，或者用绝对位置
        newFileList.add(new File("libs/framework.jar"))
        newFileList.addAll(fileSet)
        options.bootstrapClasspath = files(newFileList.toArray())
    }
}
```

4. 在 module 的 build.gradle 文件中，加入如下代码：

```
preBuild {
    doLast {
        //此处文件名根据实际情况修改：
        def imlFile = file("../.idea/modules/xxxlib/xxx.xxxlib.iml")
        try {
            def parsedXml = (new XmlParser()).parse(imlFile)
            def jdkNode = parsedXml.component[1].orderEntry.find { it.'@type' == 'jdk' }
            parsedXml.component[1].remove(jdkNode)
            def sdkString = "Android API " + android.compileSdkVersion.substring("android-".length()) + " Platform"
            new Node(parsedXml.component[1], 'orderEntry', ['type': 'jdk', 'jdkName': sdkString, 'jdkType': 'Android SDK'])
            groovy.xml.XmlUtil.serialize(parsedXml, new FileOutputStream(imlFile))
        } catch (FileNotFoundException e) {
            // nop, iml not found
            println "no iml found"
        }
    }
}
```

这个 task 在编译之前， 自动更改module.iml，将下面代码会移动最后

```
<orderEntry type="jdk" jdkName="Android API 30 Platform" jdkType="Android SDK" />
``` 

才能在编译时使用我们引入的 framework.jar。