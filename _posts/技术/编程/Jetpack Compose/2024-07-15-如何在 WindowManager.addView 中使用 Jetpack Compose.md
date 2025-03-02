---
layout: post
title: "如何在 WindowManager.addView 中使用 Jetpack Compose"
date:  2024-07-15 17:58:12 +0800
categories: ["技术", "编程", "Jetpack Compose"]
tag: ["Kotlin", "Compsoe"]
---

## 一、引出问题
Android 开发中，很常见的一个场景，通过 `WindowManager.addView()` 添加一个 View 到屏幕上。Android 最新的视图框架 Jetpack Compose，如何应用进来。这个被添加的 View 如何使用 Compose 编写视图呢？

## 二、探究问题
有的朋友肯定会马上想到使用 `ComposeView` 作为桥梁。没错，`WindowManager.addView` 方法，就接收一个 View 类型的参数。那肯定是要借助 `ComposeView` 了。但是，经过试验，直接使用 ComposeView 是行不通的。
看代码：

```
val params = WindowManager.LayoutParams(
    WindowManager.LayoutParams.WRAP_CONTENT,
    WindowManager.LayoutParams.WRAP_CONTENT,
    WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY,
    WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE,
    PixelFormat.TRANSLUCENT
)

val composeView: ComposeView = ComposeView(this).apply {
    setContent {
        Text(text = "I'm be added")
    }
}

windowManager.addView(composeView, params)
```

上面代码，编译没有问题，运行时会报错：

```
FATAL EXCEPTION: main
Process: xxxxxxxx
java.lang.IllegalStateException: ViewTreeLifecycleOwner not found from androidx.compose.ui.platform.ComposeView{8285855 V.E...... ......I. 0,0-0,0}
    at androidx.compose.ui.platform.WindowRecomposer_androidKt.createLifecycleAwareWindowRecomposer(WindowRecomposer.android.kt:352)
    at androidx.compose.ui.platform.WindowRecomposer_androidKt.createLifecycleAwareWindowRecomposer$default(WindowRecomposer.android.kt:325)
    at androidx.compose.ui.platform.WindowRecomposerFactory$Companion$LifecycleAware$1.createRecomposer(WindowRecomposer.android.kt:168)
    at androidx.compose.ui.platform.WindowRecomposerPolicy.createAndInstallWindowRecomposer$ui_release(WindowRecomposer.android.kt:224)
    at androidx.compose.ui.platform.WindowRecomposer_androidKt.getWindowRecomposer(WindowRecomposer.android.kt:300)
    at androidx.compose.ui.platform.AbstractComposeView.resolveParentCompositionContext(ComposeView.android.kt:244)
    at androidx.compose.ui.platform.AbstractComposeView.ensureCompositionCreated(ComposeView.android.kt:251)
    at androidx.compose.ui.platform.AbstractComposeView.onAttachedToWindow(ComposeView.android.kt:283)
    at android.view.View.dispatchAttachedToWindow(View.java:22065)
    at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3553)
    ...
```

看这个错误信息：
应该是从 ComposeView 中没有找到 ViewTreeLifecycleOwner， 其实很好理解。 View 的生命周期依赖于 ViewTreeLifecycleOwner， ComposeView 依赖于一个 ViewCompositonStrategy。核心问题是，ComposeView 需要一个 Lifecycle。

## 三、解决问题
有了思路自然就尝试解决问题。
首先定义一个 LifecycleOwner ，

```
import android.view.View
import androidx.lifecycle.Lifecycle
import androidx.lifecycle.LifecycleOwner
import androidx.lifecycle.LifecycleRegistry
import androidx.lifecycle.ViewModelStore
import androidx.lifecycle.ViewModelStoreOwner
import androidx.lifecycle.setViewTreeLifecycleOwner
import androidx.lifecycle.setViewTreeViewModelStoreOwner
import androidx.savedstate.SavedStateRegistry
import androidx.savedstate.SavedStateRegistryController
import androidx.savedstate.SavedStateRegistryOwner
import androidx.savedstate.setViewTreeSavedStateRegistryOwner

class MyComposeViewLifecycleOwner:
    LifecycleOwner, ViewModelStoreOwner, SavedStateRegistryOwner {

    private val lifecycleRegistry: LifecycleRegistry = LifecycleRegistry(this)
    private val savedStateRegistryController = SavedStateRegistryController.create(this)
    private val store = ViewModelStore()

    override val lifecycle: Lifecycle
        get() = lifecycleRegistry
    override val savedStateRegistry: SavedStateRegistry
        get() = savedStateRegistryController.savedStateRegistry
    override val viewModelStore: ViewModelStore
        get() = store


    fun onCreate() {
        savedStateRegistryController.performRestore(null)
        lifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_CREATE)
    }

    fun onStart() {
        lifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_START)
    }

    fun onResume() {
        lifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_RESUME)
    }

    fun onPause() {
        lifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_PAUSE)
    }

    fun onStop() {
        lifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_STOP)
    }

    fun onDestroy() {
        lifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_DESTROY)
        store.clear()
    }

    /**
     * Compose uses the Window's decor view to locate the
     * Lifecycle/ViewModel/SavedStateRegistry owners.
     * Therefore, we need to set this class as the "owner" for the decor view.
     */
    fun attachToDecorView(decorView: View?) {
        decorView?.let {
            it.setViewTreeViewModelStoreOwner(this)
            it.setViewTreeLifecycleOwner(this)
            it.setViewTreeSavedStateRegistryOwner(this)
        } ?: return
    }
}
```

再看看使用：

```
private var lifecycleOwner: MyComposeViewLifecycleOwner? = null

val params = WindowManager.LayoutParams(
    WindowManager.LayoutParams.WRAP_CONTENT,
    WindowManager.LayoutParams.WRAP_CONTENT,
    WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY,
    WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE,
    PixelFormat.TRANSLUCENT
)

val composeView: ComposeView = ComposeView(this).apply {
    setContent {
        Text(text = "I'm be added")
    }
}


// 注意，在 调用 addView 之前：
lifecycleOwner = MyComposeViewLifecycleOwner().also {
                        it.onCreate() // 注意
                        it.attachToDecorView(composeView)
                    }
windowManager.addView(composeView, params)




windowManager.removeViewImmediate(composeView)
lifecycleOwner?.onDestroy()
lifecycleOwner = null
```

OK，再次运行。成功~

But ！！！ 上面可以成功把 View 添加进去，但是没办法完成重组，重组还需以下代码：

```
val coroutineContext = AndroidUiDispatcher.CurrentThread
val runRecomposeScope = CoroutineScope(coroutineContext)
val recomposer = Recomposer(coroutineContext)
composeView.compositionContext = recomposer
runRecomposeScope.launch {
    recomposer.runRecomposeAndApplyChanges()
}
```

利用 Kotlin 的拓展方法稍微封装一下：

```
fun AbstractComposeView.addToLifecycle() {
    val viewModelStore = ViewModelStore()
    val lifecycleOwner = ComposeViewLifecycleOwner()
    lifecycleOwner.performRestore(null)
    lifecycleOwner.handleLifecycleEvent(Lifecycle.Event.ON_CREATE)
    ViewTreeLifecycleOwner.set(this, lifecycleOwner)
    ViewTreeViewModelStoreOwner.set(this) { viewModelStore }
    ViewTreeSavedStateRegistryOwner.set(this, lifecycleOwner)
    val coroutineContext = AndroidUiDispatcher.CurrentThread
    val runRecomposeScope = CoroutineScope(coroutineContext)
    val reComposer = Recomposer(coroutineContext)
    this.compositionContext = reComposer
    runRecomposeScope.launch {
        reComposer.runRecomposeAndApplyChanges()
    }
}
```

使用：

```
val composeView = SomeComposeView(context)
composeView.addToLifecycle()
windowManager.addView(composeView, layoutParam)
```