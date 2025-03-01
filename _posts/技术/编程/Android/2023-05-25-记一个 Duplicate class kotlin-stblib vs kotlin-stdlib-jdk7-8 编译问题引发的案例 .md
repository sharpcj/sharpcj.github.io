---
layout: post
title: "记一个 Duplicate class kotlin-stblib vs kotlin-stdlib-jdk7/8 编译问题引发的案例 "
date:  2023-05-25 18:58:12 +0800
categories: ["技术", "编程", "Android"]
tag: ["Android"]
---

某天将项目 kotlin 版本升级到了 1.8.0 ，然后编译报错了，
`Duplicate class kotlin-stblib vs kotlin-stdlib-jdk7/8`
然后开始寻求解决方案...

## Duplicate class kotlin-stblib vs kotlin-stdlib-jdk7/8
### kotlin-stdlib
kotlin 1.8.0 基于 JVM 1.8 编译，不再支持 JVM 1.6 和 1.7。后续不用单独在build.gradle 依赖 kotlin-stdlib-jdk7 and kotlin-stdlib-jdk8。

>If you have explicitly declared kotlin-stdlib-jdk7 and kotlin-stdlib-jdk8 as dependencies in your build scripts, then you should replace them with kotlin-stdlib.

### 解决 Duplicate class 的编译问题
- 方法一
使用 kotlin-bom 清单

```
implementation(platform("org.jetbrains.kotlin:kotlin-bom:1.8.0"))
```

- 方法二
强制把 kotlin-stdlib-jdk7 和 kotlin-stdlib-jdk8 升级到 1.8.0

```
dependencies {
    constraints {
        add("implementation", "org.jetbrains.kotlin:kotlin-stdlib-jdk7") {
            version {
                require("1.8.0")
            }
        }
        add("implementation", "org.jetbrains.kotlin:kotlin-stdlib-jdk8") {
            version {
                require("1.8.0")
            }
        }
    }
}
```

- 方法三
手动移除 kotlin-stdlib，不推荐

```
dependencies {
     implementation("com.example:lib:1.0") {
      exclude group: "org.jetbrains.kotlin", module: "kotlin-stdlib"
  }
}
```

参考文档：[https://kotlinlang.org/docs/gradle-configure-project.html#other-ways-to-align-versions](https://kotlinlang.org/docs/gradle-configure-project.html#other-ways-to-align-versions)

**kotlin-bom 库版本清单表**

- org.jetbrains.kotlin » kotlin-stdlib

- org.jetbrains.kotlin » kotlin-stdlib-jdk7

- org.jetbrains.kotlin » kotlin-stdlib-jdk8

- org.jetbrains.kotlin » kotlin-stdlib-js

- org.jetbrains.kotlin » kotlin-stdlib-common

- org.jetbrains.kotlin » kotlin-reflect

- org.jetbrains.kotlin » kotlin-osgi-bundle

- org.jetbrains.kotlin » kotlin-test

- org.jetbrains.kotlin » kotlin-test-junit

- org.jetbrains.kotlin » kotlin-test-junit5

- org.jetbrains.kotlin » kotlin-test-testng

- org.jetbrains.kotlin » kotlin-test-js

- org.jetbrains.kotlin » kotlin-test-common

- org.jetbrains.kotlin » kotlin-test-annotations-common

- org.jetbrains.kotlin » kotlin-main-kts

- org.jetbrains.kotlin » kotlin-script-runtime

- org.jetbrains.kotlin » kotlin-script-util

- org.jetbrains.kotlin » kotlin-scripting-common

- org.jetbrains.kotlin » kotlin-scripting-jvm

- org.jetbrains.kotlin » kotlin-scripting-jvm-host

- org.jetbrains.kotlin » kotlin-scripting-ide-services

- org.jetbrains.kotlin » kotlin-compiler

- org.jetbrains.kotlin » kotlin-compiler-embeddable

- org.jetbrains.kotlin » kotlin-daemon-client

具体版本对应关系：
[kotlin-bom 库版本对应表查询：https://mvnrepository.com/artifact/org.jetbrains.kotlin/kotlin-bom](https://mvnrepository.com/artifact/org.jetbrains.kotlin/kotlin-bom)