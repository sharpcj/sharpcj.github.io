---
layout: post
title: "Android 工具类 SharedPreferences 封装"
date:  2017-01-10 17:58:12 +0800
categories: ["技术", "编程", "Android"]
tag: ["Android", "SharedPreference"]
---

SharedPreferences 是 Android 数据存储方式中的一种，特别适合用来存储少量的、格式简单的数据，比如应用程序的各种配置信息，如是否打开音效，是否开启震动等等。

## SharedPreferences 存储数据的位置和格式
SharedPreferences 将数据以键值对的形式，存储在 /data/data/<package name>/shared_prefs 目录下面，以 XML 的格式保存，该 XML 文件的根元素是 <map.../>，该元素里每个子元素代表一个 key-value 对。

## 获取 SharedPreferences 的三种方式
要想使用 SharedPreferences，首先需要得到 SharedPreferences 对象。SharedPreferences 本身是一个接口，因此程序无法直接创建 SharedPreferences 的对象，Android 提供了三种获取 SharedPreferences 对象的方法。

**Context 类的 getSharedPreferences(String name, int mode) 方法**
第一个参数用于指定 SharedPreferences 的文件名，第二个参数用于指定操作模式，传入 MODE_PRIVATE 或者 0 即可，表示只有当前应用程序可以操作 SharedPreferences 文件。

**Activity 类的 getPreferences(int mode) 方法**
该方法的只接受一个参数，用于指定操作模式，和上面的方法一样。SharedPreferences 文件名以当前 Activity 的类名命名。该文件属于当前Activity私有的，只有当前Activity 可以操作。

**PreferenceManager 类的 getDefaultSharedPreferences(Context context) 方法**
这是一个静态方法，接受一个 Context 参数，以当前应用程序的报名作为前缀来命名 SharedPreferences 文件，整个应用程序可以操作。

## SharedPreferences 访问数据
**SharedPreferences 中的方法**
得到 SharedPreferences 对象之后，则可以对数据进行读的操作。SharedPreferences 中提供的公共方法中的 getXXX 方法，即为读数据，比如

getInt(String key, int defValue) 就是读取一个 int 型数据，传入第一个参数为存储数据的 key ，第二个参数为默认值，当 key 不存在的时候，会返回该默认值。

getAll() 方法，会返回 SharedPreferences 中所有的键值对。

contains(String key) 用来判断 SharedPreferences 所有键值对中是否包含键为 key 的键值对。

另外 SharedPreferences 中还有一个内部接口，`SharedPreferences.OnSharedPreferenceChangeListener` ,用来声明一个回调，当某个 SharedPreferences 改变时会回调该方法。SharedPreferences 提供了 `registerOnSharedPreferenceChangeListener(SharedPreferences.OnSharedPreferenceChangeListener listener)` 和 `unregisterOnSharedPreferenceChangeListener(OnSharedPreferenceChangeListener listener)`  分别来注册和取消注册该回调。

**SharedPreferences.Editor 中的方法**
SharedPreferences 中还有一个方法 `edit()`，返回 SharedPreferences 的另一个内部接口 `SharedPreferences.Editor` 对象，该接口提供了诸如 putXXX 方法，很容易猜到是写入数据的。同样拿 putInt(String key, int value) 来说，即往 SharedPreferences 文件中写入一个键值对。值得一提的是，调用 putXXX 方法后，还必须调用 commit() 或者apply() 方法，才能真正写入数据，这点很容易忘记（本人学习Android时曾踩过坑）。

clear() 方法，清空 SharedPreferences 中的所有键值对。

remove(String key) 方法，移除键为 key 的键值对，调用此方法后，需调用 commit 方法才能生效。

最后还有两个方法， `void apply()` 和 `boolean commit()`，它们都是真正提交某次修改的。这里主要说说这两个方法的区别：

- 从我上面写出的两个方法可以看出，apply 方法没有返回值，apply方法不会提示任何失败的提示。 commit 方法有返回值，表明修改是否提交成功。
- 从用法上来说，调用 putXXX 方法之后，调用 apply 或者 commit 方法都可以提交，而调用 remove(String key) 方法之后，需调用 commit 方法提交。
- 从本质上来说，apply是将修改数据原子提交到内存, 而后异步真正提交到硬件磁盘, 而commit是同步的提交到硬件磁盘，因此，在多个并发的提交commit的时候，他们会等待正在处理的commit保存到磁盘后在操作，从而降低了效率。而apply只是原子的提交到内容，后面有调用apply的函数的将会直接覆盖前面的内存数据，这样从一定程度上提高了很多效率。

由于在一个进程中，sharedPreference是单实例，一般不会出现并发冲突，如果对提交的结果不关心的话，建议使用apply，当然需要确保提交成功且有后续操作的话，还是需要用 commit的。下面写一个简单例子，布局文具如下：

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:paddingBottom="@dimen/activity_vertical_margin"
    android:paddingLeft="@dimen/activity_horizontal_margin"
    android:paddingRight="@dimen/activity_horizontal_margin"
    android:paddingTop="@dimen/activity_vertical_margin"
    android:orientation="vertical"
    tools:context="com.example.joy.sptest.MainActivity">

    <TextView
        android:id="@+id/tv_show"
        android:layout_marginTop="20dp"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center_horizontal"
        android:text="Hello World!" />

    <Button
        android:id="@+id/btn_write"
        android:layout_marginTop="10dp"
        android:layout_gravity="center_horizontal"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:onClick="onClick"
        android:text="写入数据"/>

    <Button
        android:id="@+id/btn_read"
        android:layout_marginTop="10dp"
        android:layout_gravity="center_horizontal"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:onClick="onClick"
        android:text="读取数据"/>

    <Button
        android:id="@+id/btn_modify"
        android:layout_marginTop="10dp"
        android:layout_gravity="center_horizontal"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:onClick="onClick"
        android:text="修改数据"/>

</LinearLayout>
```

程序代码如下：

```
package com.example.joy.sptest;

import android.content.SharedPreferences;
import android.preference.PreferenceManager;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.view.View;
import android.widget.TextView;
import android.widget.Toast;

public class MainActivity extends AppCompatActivity {

    private TextView tvShow;
    private SharedPreferences sp;
    private SharedPreferences.Editor editor;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        tvShow = (TextView) findViewById(R.id.tv_show);
    }

    public void onClick(View v) {
        sp = PreferenceManager.getDefaultSharedPreferences(MainActivity.this);
        switch (v.getId()) {
            case R.id.btn_write:
                editor = sp.edit();
                editor.putString("name", "zhangsan");
                editor.putInt("age", 18);
                if (editor.commit()) {
                    Toast.makeText(MainActivity.this, "提交成功!", Toast.LENGTH_SHORT).show();
                } else {
                    Toast.makeText(MainActivity.this, "提交失败!", Toast.LENGTH_SHORT).show();
                }
                break;
            case R.id.btn_read:
                String name = sp.getString("name", "小明");
                int age = sp.getInt("age", 0);
                tvShow.setText("学生姓名：" + name + "  学生年龄：" + age);
                break;
            case R.id.btn_modify:
                editor = sp.edit();
                editor.putString("name", "张三");
                if (editor.commit()) {
                    Toast.makeText(MainActivity.this, "提交成功!", Toast.LENGTH_SHORT).show();
                } else {
                    Toast.makeText(MainActivity.this, "提交失败!", Toast.LENGTH_SHORT).show();
                }
                break;
        }
    }
}
```

一个TextView，一开始运行会显示默认的 “Hello World!”，下面有三个button，点击写入数据，调用SharedPreferences，写入数据，点击读取数据，会从 SharedPreferences中读取数据，并显示在 textview 上，点击修改数据，则再次写入相同的 key ，替换掉之前的 value。程序运行结果如下：

![](/assets/images/技术/编程/Android/Android%20工具类%20SharedPreferences%20封装/pic1.gif)

程序运行textview显示默认的“Hello World!”，点击读取数据，因为找不到 key ，所以返回指定的默认值，点击写入数据，写入姓名"zhangsan"，年龄“18”。点击修改数据，将姓名       改为“张三”。
通过DDMS文件可以查看,

![](/assets/images/技术/编程/Android/Android%20工具类%20SharedPreferences%20封装/pic2.png)

在 `data/data/com.example.joy.sptest/shared_prefs` 目录下面生成了一个 XML 文件：`com.example.joy.sptest_preferences.xml`。导出该文件，打开可以看到内容如下：

![](/assets/images/技术/编程/Android/Android%20工具类%20SharedPreferences%20封装/pic3.png)

最后贴出我自己封装后常用的读写 SharedPreferences 的方法：

```
public static void putSharedPreferences(Context context, String key, Object value) {
        String type = value.getClass().getSimpleName();
        SharedPreferences sharedPreferences = PreferenceManager.getDefaultSharedPreferences(context);
        SharedPreferences.Editor editor = sharedPreferences.edit();
        if ("Integer".equals(type)) {
            editor.putInt(key, (Integer) value);
        } else if ("Boolean".equals(type)) {
            editor.putBoolean(key, (Boolean) value);
        } else if ("String".equals(type)) {
            editor.putString(key, (String) value);
        } else if ("Float".equals(type)) {
            editor.putFloat(key, (Float) value);
        } else if ("Long".equals(type)) {
            editor.putLong(key, (Long) value);
        }
        editor.commit();
    }

    public static Object getSharedPreferences(Context context, String key, Object defValue) {
        String type = defValue.getClass().getSimpleName();
        SharedPreferences sharedPreferences = PreferenceManager.getDefaultSharedPreferences(context);
        //defValue为为默认值，如果当前获取不到数据就返回它
        if ("Integer".equals(type)) {
            return sharedPreferences.getInt(key, (Integer) defValue);
        } else if ("Boolean".equals(type)) {
            return sharedPreferences.getBoolean(key, (Boolean) defValue);
        } else if ("String".equals(type)) {
            return sharedPreferences.getString(key, (String) defValue);
        } else if ("Float".equals(type)) {
            return sharedPreferences.getFloat(key, (Float) defValue);
        } else if ("Long".equals(type)) {
            return sharedPreferences.getLong(key, (Long) defValue);
        }
        return null;
    }
```


