---
layout: post
title: "Jetpack Compose 中如何实现全面屏"
date:  2024-07-16 17:58:12 +0800
categories: ["技术", "编程", "Jetpack Compose"]
tag: ["Kotlin", "Compsoe"]
---

看问题本质，设置全面屏，是系统窗口的行为，与 View 和 Compose 有什么关系呢？
所以，原理和传统 View 视图是一样的，甚至 Api 都是一模一样的，不熟悉的可以看我之前的文章。传送门：

[Android 全面屏体验](https://www.cnblogs.com/joy99/p/15551483.html)

那为什么还要写这篇文章呢？主要是在 Compose 中写法上的一些区别，直接上代码：

找到主题设置的代码，默认生成的主题如下：

```
@Composable
fun HelloComposeTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    // Dynamic color is available on Android 12+
    dynamicColor: Boolean = true,
    content: @Composable () -> Unit
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val context = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(context) else dynamicLightColorScheme(context)
        }

        darkTheme -> DarkColorScheme
        else -> LightColorScheme
    }
    val view = LocalView.current
    if (!view.isInEditMode) {
        SideEffect {
            val window = (view.context as Activity).window
            window.statusBarColor = colorScheme.primary.toArgb()
            WindowCompat.getInsetsController(window, view).isAppearanceLightStatusBars = darkTheme
        }
    }

    MaterialTheme(
        colorScheme = colorScheme,
        typography = Typography,
        content = content
    )
}
```

与系统状态栏，导航栏相关的代码：

![](/assets/images/技术/编程/Jetpack%20Compose/compose%20全面屏/pic1.png)

修改如下：

```
val view = LocalView.current
    if (!view.isInEditMode) {
        SideEffect {
            val window = (view.context as Activity).window
            // window.statusBarColor = colorScheme.primary.toArgb()
            // 设置状态栏和导航栏透明
            window.statusBarColor = Color.TRANSPARENT
            window.navigationBarColor = Color.TRANSPARENT
            
            // 让内容可以显示到状态栏和导航栏下面区域
            WindowCompat.setDecorFitsSystemWindows(window, false)

            // light 主题显示深色，dark 主题显示浅色，强制显示深色，就设为 true, 前置显示浅色就设为 false
            WindowCompat.getInsetsController(window, view).isAppearanceLightStatusBars = darkTheme 

            // 导航栏默认会有一层半透明遮罩， 这行代码，去掉默认半透明遮罩
            window.isNavigationBarContrastEnforced = false
        }
    }
```

还剩一个问题，大多数时候，我们虽然希望内容延伸到状态栏和导航栏下面，但是并不希望内容被系统栏挡住，只是想要达到背景颜色保持一致的效果。解决思路就是顶层 View 设置了背景颜色，然后给它上下流出 padding， 其中 top padding 即为状态栏高度，bottom padding 为导航栏的高度。

传统View视图有两种解决方案：

- 动态设置。新的 api 获取 statusBarHeight 和 navigationBarHeight，需要通过在 Activity 中注册监听，在回调中获取。这个回调时机可能晚于一些初始化工作，所以，只能在回调中动态设置。
- 在 xml 文件中，根布局设置 `fitsSystemWindows` 属性为 true。

最后来说说，Compose 中如何设置呢？
Compose Modifier 中提供了一个 api —— `Modifier.systemBarsPadding()` 即可。