---
layout: post
title: "从最简单的HelloWorld理解MVP模式"
date:  2016-11-30 17:58:12 +0800
categories: ["技术", "编程", "Android"]
tag: ["Android", "架构设计", "MVP"]
---

大多数编程语言相关的学习书籍，都会以hello,world这个典型的程序作为第一个示例。作为Android应用开发者，无论使用eclipse还是用android studio，在新建项目的时候，一直按IDE默认选择项，下一步进行下去，就会创建出一个可以运行的hello,world应用程序。对于这个程序，可以认为是采用MVC模式，对应关系为：
- View：对应于布局文件
- Model：业务逻辑和实体模型
- Controller：对应于Activity

但是数据绑定、事件处理（hello world程序没有）的代码都在Activity中，Activity看起来既担任了View的角色，又担任了Controller的角色。这样随着程序业务逻辑越来越复杂，Activity中的代码就会越来越多，最终结果就是程序的耦合度越来越高，程序修改和维护越来越难。于是MVP模式的优点就显示出来了。下面我就以这个最简单的程序，来谈谈我对mvp模式的理解。
先上代码：

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context="com.example.joy.etest.MainActivity">

    <TextView
        android:id="@+id/tv_show"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Hello World!" />

    <Button
        android:id="@+id/btn_test"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="测试"/>

</LinearLayout>
```

布局文件很简单，一个 TextView ,程序运行显示 “Hello World!”。这里我增加了一个Button，点击Button，开始倒计时10秒钟，以此来模拟一些耗时的操作，10秒钟后，TextView显示"Hello MVP!"。再看java代码，在MVC模式下，我们直接在 Activity 中通过 setText 方法，就可以给 TextView 设置显示的内容。MVP模式相对于MVC模式来说，就是将Controller这部分从Activity中分离出来，让Activity只担任View的角色，View和Model之间的桥梁作用由 presenter 来承担，从而达到解耦的目的。具体的方法就是通过接口回调来实现。下面是时候展现真正的步骤了：

首先定义一个接口,就叫 IShowView吧，里面有一个 show 方法,用于给TextView设置显示内容

```
package com.example.joy.mvptest;

public interface IShowView {
    void show(String str);
}
```

MainActivity实现上述接口：

```
package com.example.joy.mvptest;

import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.view.View;
import android.widget.Button;
import android.widget.TextView;

public class MainActivity extends AppCompatActivity implements IShowView, View.OnClickListener {

    private TextView mTvShow;
    private Button mBtnTest;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mTvShow = (TextView) findViewById(R.id.tv_show);
        mBtnTest = (Button) findViewById(R.id.btn_test);
        mBtnTest.setOnClickListener(this);

    }

    @Override
    public void show(String str) {
        mTvShow.setText(str);
    }

    @Override
    public void onClick(View v) {
        if(v.getId() == R.id.btn_test) {

        }
    }
}
```

代码很简单。MainActivity实现接口中的show方法，即为TextView赋值。Button的点击事件暂时没有写。

其次，presenter模块也要定义一个接口，与 MainActivity 实现的接口提供类似的方法，就叫 IPresenter 吧：

```
package com.example.joy.mvptest.presenter;

public interface IPresenter {
    void show();
}
```

注意，这个接口里的 show 方法没有参数，和前面 IShowView 中的方法签名不一样，当然，你也可以不命名为 show ，方法名可以自己随便起。这里的方法参数完全是根据实际需要来确定。有了接口，当然还需要有接口的实现类，命名为 PresenterComl ：

```
package com.example.joy.mvptest.presenter;

import android.os.AsyncTask;
import android.os.SystemClock;
import android.util.Log;

import com.example.joy.mvptest.IShowView;

public class PresenterComl implements IPresenter {
    private IShowView iShowView;

    public PresenterComl(IShowView iShowView) {
        this.iShowView = iShowView;
    }

    @Override
    public void show() {
        new AsyncTask<Void, Void, Void>() {

            @Override
            protected void onPostExecute(Void aVoid) {
                super.onPostExecute(aVoid);
                Log.d("joy99", "下载完成。");
                iShowView.show("Hello,MVP!");
            }

            @Override
            protected Void doInBackground(Void... params) {
                for (int i = 0; i < 10; i++) {
                    Log.d("joy99", "正在下载...预计剩余时间 " + (10 - i) + "秒");
                    SystemClock.sleep(1000);
                }
                return null;
            }
        }.execute();

    }
}
```

此处我用了一个异步任务。将倒计时打印出来，模拟一些耗时的数据操作，比如网络请求等等。该类中声明了一个IShowView接口的实现类的对象，iShowView，倒计时完成，调用iShowView的show方法，将“结果”传递过去，典型的接口回调的用法。

剩下的最后一步就很清晰了，在 MainActivity 中定义 PresenterComl 类的对象 iPresenter，然后，在Button的点击事件中，调用 iPresenter 的show方法。注意构造方法中MainActivity本身，作为实现 IShowView 的类的对象传递进去。修改后的 MainActivity 类代码如下：

```
package com.example.joy.mvptest;

import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.view.View;
import android.widget.Button;
import android.widget.TextView;

import com.example.joy.mvptest.presenter.IPresenter;
import com.example.joy.mvptest.presenter.PresenterComl;

public class MainActivity extends AppCompatActivity implements IShowView, View.OnClickListener {

    private TextView mTvShow;
    private Button mBtnTest;

    private IPresenter iPresenter;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mTvShow = (TextView) findViewById(R.id.tv_show);
        mBtnTest = (Button) findViewById(R.id.btn_test);
        mBtnTest.setOnClickListener(this);

        iPresenter = new PresenterComl(this);
    }

    @Override
    public void show(String str) {
        mTvShow.setText(str);
    }

    @Override
    public void onClick(View v) {
        if(v.getId() == R.id.btn_test) {
            iPresenter.show();
        }
    }
}
```

OK,通过这样一个小程序将MVP模式分析了一下，它的本质其实就是接口回调。当然，对于这样一个小的不能再小的程序来说，用MVP模式，确实看起来更复杂了。但是代码逻辑复杂了，MVP模式的优势就显示出来了。最后总结一下：

　　**使用MVP模式会使得代码多出一些接口，但是使得代码逻辑更加清晰，尤其是在处理复杂界面和逻辑时，我们可以对同一个activity将每一个业务都抽离成一个Presenter，这样代码既清晰、逻辑明确又方便我们扩展。当然如果我们的业务逻辑本身就比较简单的话使用MVP模式就显得没那么必要。所以我们不需要为了用它而用它，具体的还是要要业务需要。**