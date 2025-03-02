---
layout: post
title: "Jetapck Compose 去除点击水波纹效果"
date:  2024-07-17 17:58:12 +0800
categories: ["技术", "编程", "Jetpack Compose"]
tag: ["Kotlin", "Compsoe"]
---

问题：Jetpack Compose 中使用 Material 包中的控件，点击默认会有水波纹效果。如何去除这个点击水波纹效果呢？
看下 Modifier.clickable 的签名：

```
fun Modifier.clickable(
    interactionSource: MutableInteractionSource,
    indication: Indication?,
    enabled: Boolean = true,
    onClickLabel: String? = null,
    role: Role? = null,
    onClick: () -> Unit
)
```

其实就是 `indication` 这个参数决定的。

针对局部单个点击去除水波纹效果，我们只需要将该参数设置为 null 即可。
针对全局如何设置呢？ `indication` 参数默认值是 `indication = LocalIndication.current`。显然这是通过 CompositionLocal 的方式传递下来的。那么我们只需要自己定义一个 Indication，在上层把这个属性值给替换掉即可。比如需要对整个 Activity 生效，就在该 Activity 的根组合项中替换，如果需要整个应用生效，可以在主题中进行替换。

先看一下默认的 Indication 是如何实现的：

```
val LocalIndication = staticCompositionLocalOf<Indication> {
    DefaultDebugIndication
}

private object DefaultDebugIndication : Indication {

    private class DefaultDebugIndicationInstance(
        private val isPressed: State<Boolean>,
        private val isHovered: State<Boolean>,
        private val isFocused: State<Boolean>,
    ) : IndicationInstance {
        override fun ContentDrawScope.drawIndication() {
            drawContent()
            if (isPressed.value) {
                drawRect(color = Color.Black.copy(alpha = 0.3f), size = size)
            } else if (isHovered.value || isFocused.value) {
                drawRect(color = Color.Black.copy(alpha = 0.1f), size = size)
            }
        }
    }

    @Composable
    override fun rememberUpdatedInstance(interactionSource: InteractionSource): IndicationInstance {
        val isPressed = interactionSource.collectIsPressedAsState()
        val isHovered = interactionSource.collectIsHoveredAsState()
        val isFocused = interactionSource.collectIsFocusedAsState()
        return remember(interactionSource) {
            DefaultDebugIndicationInstance(isPressed, isHovered, isFocused)
        }
    }
}
```

其实在这个文件中已经实现了一个 NoIndication, 只不过是 private 的：

```
private object NoIndication : Indication {
    private object NoIndicationInstance : IndicationInstance {
        override fun ContentDrawScope.drawIndication() {
            drawContent()
        }
    }

    @Composable
    override fun rememberUpdatedInstance(interactionSource: InteractionSource): IndicationInstance {
        return NoIndicationInstance
    }
}
```

这里面涉及到了两个接口：

```
@Stable
interface Indication {

    @Composable
    fun rememberUpdatedInstance(interactionSource: InteractionSource): IndicationInstance
}

interface IndicationInstance {

    fun ContentDrawScope.drawIndication()
}
```

看起来，逻辑很清晰，自定义一个 Indication 需要两步：

1. 实现IndicationInstance 接口，实现 ContentDrawScope.drawIndication() 方法。并把这个对象返回。看方法名，应该就是绘制点击效果。默认实现中使用到了 interactionSource ，根据点击的状态不同，绘制出不同的效果。
2. 实现 Indication 的接口，实现 rememberUpdatedInstance 方法，返回上面的 IndicationInstance 类型的对象。

有了理论基础，我们实操一下，去掉点击水波纹效果，也分为两步：

1. 定义一个无点击效果的 Indication。直接把源码中的那个私有的照抄就行了，让我们应用可用。

```
object NoIndication: Indication {
    private object NoIndicationInstance: IndicationInstance {
        override fun ContentDrawScope.drawIndication() {
            drawContent()
        }

    }

    @Composable
    override fun rememberUpdatedInstance(interactionSource: InteractionSource): IndicationInstance {
        return NoIndicationInstance
    }

}
```

2. 应用这个 Indication

```
@Composable
fun CustomTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    // Dynamic color is available on Android 12+
    dynamicColor: Boolean = true,
    content: @Composable () -> Unit
) {
    ...
    MaterialTheme(
        colorScheme = colorScheme,
        typography = Typography,
        content = {
            CompositionLocalProvider(LocalIndication provides NoIndication) {
                content()
            }
        }
    )
}
```

Done ！！！

相信有了这个理论基础，我们完全可以自定义自己想要的点击效果了。