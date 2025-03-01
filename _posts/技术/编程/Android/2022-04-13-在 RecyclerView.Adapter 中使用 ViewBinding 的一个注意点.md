---
layout: post
title: "在RecyclerView.Adapter中使用 ViewBinding 的一个注意点"
date:  2022-04-13 18:58:12 +0800
categories: ["技术", "编程", "Android"]
tag: ["Android", "ViewBinding"]
---

使用 viewpager2 时遇到如下错误， 使用 recyclerview 也有可能会遇到 ：

```
2022-02-10 14:15:43.510 12151-12151/com.sharpcj.demo1 D/sharpcj_tag: onBindViewHolder
...
2022-02-10 14:15:43.548 12151-12151/com.sharpcj.demo1 D/AndroidRuntime: Shutting down VM
2022-02-10 14:15:43.550 12151-12151/com.sharpcj.demo1 E/AndroidRuntime: FATAL EXCEPTION: main
    Process: com.sharpcj.demo1, PID: 12151
    java.lang.IllegalStateException: Pages must fill the whole ViewPager2 (use match_parent)
    ...
```

原因在日志中能看出来，就是 adapter 的 item 必须设置为 `match_parent`。

例如，我这个 demo 中，使用 viewpager2 实现一个 banner 页面， item view 只有一个 ImageView 用来展示图片，我的 banner 页面高度为 200dp。我应该将 viewpager2 的高度设置为 200dp，而 item 布局文件，高度也要设置为 `match_parent`。

banner_item_view.xml 文件如下：

```
<?xml version="1.0" encoding="utf-8"?>
<ImageView xmlns:android="http://schemas.android.com/apk/res/android"
   android:id="@+id/iv_banner"
   android:layout_width="match_parent"
   android:layout_height="match_parent">
</ImageView>
```

adpater 部分代码如下：

```
override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): MyViewHolder {
    val inflate = LayoutInflater.from(parent.context).inflate(R.layout.banner_item_view, parent, false)
    return MyViewHolder(inflate)
}
```

而如果使用 ViewBinding ，则代码如下：

```
override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): MyViewHolder {
    val viewBinding = BannerItemViewBinding.inflate(LayoutInflater.from(parent.context))
    val layoutParams = ViewGroup.LayoutParams(
        ViewGroup.LayoutParams.MATCH_PARENT,
        ViewGroup.LayoutParams.MATCH_PARENT
    )
    viewBinding.root.layoutParams = layoutParams
    return MyViewHolder(viewBinding.root)
}
```

此时，布局文件中设置的宽高已经不生效了。

总结：

>如果不使用 ViewBinding， 则须在 xml 文件中设置 item view 宽高为 `match_parent`
>如果使用 ViewBinding， 则需要动态设置 layoutParams, 且宽高设置为 `LayoutParams.MATCH_PARENT`