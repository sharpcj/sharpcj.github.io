---
layout: post
title: "FloatingActionButton的一点学习感悟"
date:  2016-07-12 17:58:12 +0800
categories: ["技术", "编程", "Android"]
tag: ["Android", "FloatingActionButton"]
---

最近在学习android材料设计的新控件，前面一篇文章讲到 CoordinatorLayout 结合几个新控件可以实现的几个效果。其中第一个是，Coordinatorlayout + FloatingActionButton，配合使用，当弹出 Snackbar 的时候，FloatingActionBar会跟随上移和下移。这次再针对 FloatingActionButton 具体分析一下。先贴出布局文件和java代码：

```
<?xml version="1.0" encoding="utf-8"?>
<android.support.design.widget.CoordinatorLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <Button
        android:id="@+id/btnSnackBar"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="点击弹出SnackBar"/>


    <android.support.design.widget.FloatingActionButton
        android:id="@+id/floatingActionBar"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="bottom|right"
        android:src="@mipmap/ic_launcher" />
    
</android.support.design.widget.CoordinatorLayout>
```

```
public class MainActivity extends Activity {
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_sixth);
        ((Button) findViewById(R.id.btnSnackBar)).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Snackbar.make(v,"这是一个snackBar",Snackbar.LENGTH_SHORT).show();
            }
        });
    }
}
```

代码很简洁，效果如下：
![](/assets/images/技术/编程/Android/FloatingActionButton的一点学习感悟/pic1.gif)

这个效果只有在 `CoordinatorLayout` 下，`FloatingActionButton` 配合 `Snackbar` 使用才会有。

我们知道，`FloatingActionButton` 间接继承自 ImagButton, 如果将 FloatingActionButton 换成常用的 ImagButton 或者 ImageView，效果将不存在。为什么会这样呢。在 FloatingActionButton 的定义中发现，通过反射，利用注解注入了一个 behavior 属性。FloatingActionButton 类中定义了一个内部类——Behaivor,继承自 `CoordinatorLayout.Behavior<FloatingActionButton>` 。里面有两个方法如下：

```
@Override
public boolean layoutDependsOn(CoordinatorLayout parent,FloatingActionButton child, View dependency) {
    // We're dependent on all SnackbarLayouts (if enabled)
    return SNACKBAR_BEHAVIOR_ENABLED && dependency instanceof Snackbar.SnackbarLayout;
}

@Override
public boolean onDependentViewChanged(CoordinatorLayout parent, FloatingActionButton child,View dependency) {
    if (dependency instanceof Snackbar.SnackbarLayout) {
        updateFabTranslationForSnackbar(parent, child, dependency);
    } else if (dependency instanceof AppBarLayout) {
        // If we're depending on an AppBarLayout we will show/hide it automatically
        // if the FAB is anchored to the AppBarLayout
        updateFabVisibility(parent, (AppBarLayout) dependency, child);
    }
    return false;
}

private float getFabTranslationYForSnackbar(CoordinatorLayout parent,FloatingActionButton fab) {
    float minOffset = 0;
    final List<View> dependencies = parent.getDependencies(fab);
    for (int i = 0, z = dependencies.size(); i < z; i++) {
    　　final View view = dependencies.get(i);
    　　if (view instanceof Snackbar.SnackbarLayout && parent.doViewsOverlap(fab, view)) {
            minOffset = Math.min(minOffset,ViewCompat.getTranslationY(view) - view.getHeight());
        }
    }
    return minOffset;
}
```

layoutDependsOn 方法，判断底下弹出的是不是snackbar，如果是，floatingactionbutton才会联动。

当需要联动的snackbar向上移动的过程中，会不断调用 onDependentViewChanged 方法。

getFabTranslationYForSnackbar 用来获取SnackBar逐渐弹出来的时候变化的高度。

明白了这三个方法之后，我们就可以对这个效果进行修改写。下面我们自己写一个类，让其继承自 CoordinatorLayout.Behavior<FloatingActionButton>，让FloatingActionButton在上升过程中实现随高度变化旋转效果。

代码如下:

```
public class MyBehavior extends CoordinatorLayout.Behavior<FloatingActionButton> {
    /**
     * Default constructor for instantiating Behaviors.
     */
    public MyBehavior() {
    }

    /**
     * Default constructor for inflating Behaviors from layout. The Behavior will have
     * the opportunity to parse specially defined layout parameters. These parameters will
     * appear on the child view tag.
     *
     * @param context
     * @param attrs
     */
    public MyBehavior(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    /**
     * 判断底下弹出的是不是snackbar，如果是，floatingactionbutton才会联动
     *
     * @param parent
     * @param child
     * @param dependency
     * @return
     */
    @Override
    public boolean layoutDependsOn(CoordinatorLayout parent,
                                   FloatingActionButton child, View dependency) {

        return dependency instanceof Snackbar.SnackbarLayout;
    }


    /**
     * 当需要联动的snackbar向上移动的过程中，会不断调用这个方法
     *
     * @param parent
     * @param child
     * @param dependency
     * @return
     */
    @Override
    public boolean onDependentViewChanged(CoordinatorLayout parent, FloatingActionButton child,
                                          View dependency) {

        float fTransY = getFabTranslationYForSnackbar(parent, child);

        float iRate = -fTransY / dependency.getHeight();

        child.setRotation(iRate * 90);

        child.setTranslationY(fTransY);

        return false;
    }


    /**
     * 用来获取SnackBar逐渐弹出来的时候变化的高度
     *
     * @param parent
     * @param fab
     * @return
     */
    private float getFabTranslationYForSnackbar(CoordinatorLayout parent,
                                                FloatingActionButton fab) {
        float minOffset = 0;
        final List<View> dependencies = parent.getDependencies(fab);
        for (int i = 0, z = dependencies.size(); i < z; i++) {
            final View view = dependencies.get(i);
            if (view instanceof Snackbar.SnackbarLayout && parent.doViewsOverlap(fab, view)) {
                minOffset = Math.min(minOffset,
                        ViewCompat.getTranslationY(view) - view.getHeight());
            }
        }
        return minOffset;
    }
}
```

然后修改布局文件，为 FloatingActionButton 设置 layout_behavior 属性为自定义的MyBehavior，

```
<?xml version="1.0" encoding="utf-8"?>
<android.support.design.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    xmlns:app="http://schemas.android.com/apk/res-auto">

    <Button
        android:id="@+id/btnSnackBar"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="点击弹出SnackBar"/>


    <android.support.design.widget.FloatingActionButton
        android:id="@+id/floatingActionBar"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="bottom|right"
        app:layout_behavior="com.example.joy.coordinatorlayouttest.MyBehavior"
        android:src="@mipmap/ic_launcher" />

</android.support.design.widget.CoordinatorLayout>
```

效果如下：

![](/assets/images/技术/编程/Android/FloatingActionButton的一点学习感悟/pic2.gif)