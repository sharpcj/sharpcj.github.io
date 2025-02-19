---
layout: post
title: "ActionMode 就记这么一点，不能更多了"
date:  2017-02-07 18:58:12 +0800
categories: ["技术", "编程", "Android"]
tag: ["Android", "ActionMode"]
---

话说程序猿都是段子手，看到有的程序猿写文章，前面都会先写一个段子，我这么有幽默感的段子手，也决定效仿一下。

“段子。”　　

写完段子，下面开始进入正题。

![](/assets/images/技术/编程/Android/ActionMode%20就记这么一点，不能更多了/pic1.jpg)

今天要说的 ActionMode 是 Android 提供的一种实现菜单方式。Android 实现菜单的方式有很多种，比如 ActionBar、ToolBar、ContextMenu、OptionMenu、PopupMenu、PopupWindow 等等。ActionMode 自己之前一直没用过，最近接手一个系统 app，里面有用这个，学习了下它的实现方式。ActionMode 是临时占据了ActionBar的位置，使用起来也很简单。先看下效果图：

![](/assets/images/技术/编程/Android/ActionMode%20就记这么一点，不能更多了/pic2.gif)

程序一运行，有一个 TextView 显示 Hello World! ，为了作对比，我给出了一个 OptionMenu, 点击 菜单选项，弹出相应的 Toast 。 点击 Hello World，进入 ActionMode 模式，可以看到， ActionMode 临时占用了 ActionBar ， 点击相应的 菜单选项，这里我也只是弹出相应的 Toast 以作演示。

下面贴出代码，布局文件如下：

```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/activity_main"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:paddingBottom="@dimen/activity_vertical_margin"
    android:paddingLeft="@dimen/activity_horizontal_margin"
    android:paddingRight="@dimen/activity_horizontal_margin"
    android:paddingTop="@dimen/activity_vertical_margin"
    tools:context="com.example.joy.actionmodetest.MainActivity">

    <TextView
        android:id="@+id/tv_test"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Hello World!" />
</RelativeLayout>
```

在 res/menu 下面新建 OptionMenu 的布局文件如下：

```
<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto">
    <item android:id="@+id/op_test1" android:icon="@mipmap/ic_launcher" android:title="menu1"
        app:showAsAction="always"/>
    <item android:id="@+id/op_test2" android:icon="@mipmap/ic_launcher" android:title="menu2"
        app:showAsAction="always"/>
</menu>
```

同时还要在 res/menu 下面 新建一个xml 文件，作为 ActionMode 菜单的布局文件，如下：

```
<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto">
    <item android:id="@+id/am_test1" android:icon="@mipmap/remind" android:title="menu1"
        app:showAsAction="always"/>
    <item android:id="@+id/am_test2" android:icon="@mipmap/favorites" android:title="menu2"
        app:showAsAction="always"/>
</menu>
```

java 代码如下：

```
package com.example.joy.actionmodetest;

import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.view.ActionMode;
import android.view.Menu;
import android.view.MenuInflater;
import android.view.MenuItem;
import android.view.View;
import android.widget.TextView;
import android.widget.Toast;

public class MainActivity extends AppCompatActivity {

    private TextView tv_test;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        tv_test = (TextView) findViewById(R.id.tv_test);
        tv_test.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                startActionMode(new MyCallback());
            }
        });
    }

    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        MenuInflater menuInflater = getMenuInflater();
        menuInflater.inflate(R.menu.option_menu,menu);
        return true;
    }

    @Override
    public boolean onOptionsItemSelected(MenuItem item) {
        switch(item.getItemId()){
            case R.id.op_test1:
                Toast.makeText(MainActivity.this,"option-menu1-test",Toast.LENGTH_SHORT).show();
                break;
            case R.id.op_test2:
                Toast.makeText(MainActivity.this,"option-menu2-test",Toast.LENGTH_SHORT).show();
                break;
            default:
                break;
        }
        return true;
    }

    private class MyCallback implements ActionMode.Callback {
        @Override
        public boolean onCreateActionMode(ActionMode mode, Menu menu) {
            mode.getMenuInflater().inflate(R.menu.action_mode_menu, menu);
            return true;
        }

        @Override
        public boolean onPrepareActionMode(ActionMode mode, Menu menu) {
            return false;
        }

        @Override
        public boolean onActionItemClicked(ActionMode mode, MenuItem item) {
            switch (item.getItemId()) {
                case R.id.am_test1:
                    Toast.makeText(MainActivity.this,"warning!",Toast.LENGTH_SHORT).show();
                    break;
                case R.id.am_test2:
                    Toast.makeText(MainActivity.this,"favourite!",Toast.LENGTH_SHORT).show();
                    break;
                default:
                    break;
            }
            mode.finish();
            return true;
        }

        @Override
        public void onDestroyActionMode(ActionMode mode) {

        }
    }
}
```

ActionMode 的使用非常简单，调用 Activity 的 startActionMode 方法进入 ActionMode 模式即可，在不需要使用的时候调用 ActionMode 的 finish 方法退出 ActionMode 模式。startActionMode 方法接收一个 ActionMode.Callback 的参数，使用 ActionMode 的关键在于实现这个接口中 ActionMode 在各个声明周期的回调方法：

- **onCreateActionMode(ActionMode, Menu)**
在初始创建的时候调用
- **onPrepareActionMode(ActionMode, Menu)**
在创建之后准备绘制的时候调用
- **onActionItemClicked(ActionMode, Menu)**
当点击 ActionMode 菜单选项的时候调用
- **onDestroyActionMode(ActionMode)**
当退出 ActionMode 的时候调用

ActionMode 的 getMenuInflater 方法返回一个 MenuInflater 对象，用来加载 ActionMode 的菜单布局文件。现在这个 ActionMode 黑色背景，还带点红色底线，样式很丑，自己都快看不下去了。那么怎么设置 ActionMode 的样式呢？答案是通过 style 来设置。打开 res/values/styles.xml 文件进行修改。具体可以修改如下选项，每个选项的意义通过 name 属性也都很清楚的体现出来：

```
<item name="actionModeStyle">@style/Widget.AppCompat.ActionMode</item>
<item name="actionModeBackground">@drawable/abc_cab_background_top_material</item>
<item name="actionModeSplitBackground">?attr/colorPrimaryDark</item>
<item name="actionModeCloseDrawable">@drawable/abc_ic_ab_back_mtrl_am_alpha</item>
<item name="actionModeCloseButtonStyle">@style/Widget.AppCompat.ActionButton.CloseMode</item>
<item name="actionModeCutDrawable">@drawable/abc_ic_menu_cut_mtrl_alpha</item>
<item name="actionModeCopyDrawable">@drawable/abc_ic_menu_copy_mtrl_am_alpha</item>
<item name="actionModePasteDrawable">@drawable/abc_ic_menu_paste_mtrl_am_alpha</item>
<item name="actionModeSelectAllDrawable">@drawable/abc_ic_menu_selectall_mtrl_alpha</item>
<item name="actionModeShareDrawable">@drawable/abc_ic_menu_share_mtrl_alpha</item>
```

好了。ActionMode 就记这么一点，不能更多了...