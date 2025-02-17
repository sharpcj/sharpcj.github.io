---
layout: post
title: "Android子线程更新UI的方法总结"
date:  2016-12-02 17:58:12 +0800
categories: ["技术", "编程", "Android"]
tag: ["Android", "ANR", "UI线程"]
---

消息机制，对于Android开发者来说，应该是非常熟悉。对于处理有着大量交互的场景，采用消息机制，是再好不过了。有些特殊的场景，比如我们都知道，在Android开发中，子线程不能更新UI,而主线程又不能进行耗时操作，一种常用的处理方法就是，在子线程中进行耗时操作，完成之后发送消息，通知主线程更新UI。或者使用异步任务，异步任务的实质也是对消息机制的封装。

关于子线程到底能不能更新UI这个问题，之前看到一篇文章很有趣，让我对这个问题也有了新的认识，那么我也来写个简单例子测试下，布局文件如下:

```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:paddingBottom="@dimen/activity_vertical_margin"
    android:paddingLeft="@dimen/activity_horizontal_margin"
    android:paddingRight="@dimen/activity_horizontal_margin"
    android:paddingTop="@dimen/activity_vertical_margin"
    tools:context="com.example.joy.messagetest.MainActivity">

    <TextView
        android:id="@+id/tv_test"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_centerHorizontal="true"
        android:text="Hello World!" />

</RelativeLayout>
```

布局中只有一个TextView，java代码如下：

```
package com.example.joy.messagetest;

import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.view.View;
import android.widget.TextView;

public class MainActivity extends AppCompatActivity {

    private TextView mTvTest;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        initView();

        new Thread(new Runnable() {
            @Override
            public void run() {
                mTvTest.setText("子线程可以更新UI");
            }
        }).start();
    }

    private void initView() {
        mTvTest = (TextView) findViewById(R.id.tv_test);
    }
}
```

代码也很简单，我开启子线程，在子线程中，将 TextView 内容设置为“子线程可以更新UI”,而在布局文件中，TextView 的 text 为“Hello world！”，那么现在运行程序，可能会出现的结果有三种：
- 程序崩了，抛异常了：说明子线程不能更新UI
- 程序正常运行，textview 上面显示“Hello World！”：说明子线程不能更新UI
- 程序正常运行，textview 上面显示“子线程可以更新UI”:说明子线程可以更新UI

运行程序，结果如下：

![](/assets/images/技术/编程/Android/Android子线程更新UI的方法/pic1.png)

这说明什么？从结果看，子线程更新UI成功了。真的是这样吗？我自己也不相信，赶紧再验证一遍。这次我在布局文件中添加一个Button，修改后的布局文件如下：

```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:paddingBottom="@dimen/activity_vertical_margin"
    android:paddingLeft="@dimen/activity_horizontal_margin"
    android:paddingRight="@dimen/activity_horizontal_margin"
    android:paddingTop="@dimen/activity_vertical_margin"
    tools:context="com.example.joy.messagetest.MainActivity">

    <TextView
        android:id="@+id/tv_test"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_centerHorizontal="true"
        android:text="Hello World!" />

    <Button
        android:id="@+id/btn_test1"
        android:layout_below="@id/tv_test"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_centerHorizontal="true"
        android:text="子线程更新UI测试"/>

</RelativeLayout>
```

同时修改java代码:

```
package com.example.joy.messagetest;

import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.view.View;
import android.widget.Button;
import android.widget.TextView;

public class MainActivity extends AppCompatActivity implements View.OnClickListener {

    private TextView mTvTest;
    private Button mBtnTest1;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        initView();

        new Thread(new Runnable() {
            @Override
            public void run() {
                mTvTest.setText("子线程可以更新UI");
            }
        }).start();
    }

    private void initView() {
        mTvTest = (TextView) findViewById(R.id.tv_test);
        mBtnTest1 = (Button) findViewById(R.id.btn_test1);
        mBtnTest1.setOnClickListener(this);
    }

    @Override
    public void onClick(View v) {
        switch(v.getId()){
            case R.id.btn_test1:
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        mTvTest.setText("子线程真的可以更新UI吗？");
                    }
                }).start();
                break;
            default:
                break;
        }
    }
}
```

我们增加了一个button，点击button，启动一个子线程，在子线程中将 textview 的显示内容改为 “子线程真的可以更新UI吗？”。同样按照前面的分析，我们再来验证一下。重新运行程序， textview 显示 “子线程可以更新UI”, 然后我们点击 button。结果如下：

![](/assets/images/技术/编程/Android/Android子线程更新UI的方法/pic2.gif)

怎么回事？程序崩了。仔细看，你会发现，点击 button 后 textview 的内容其实是发生了更改的，然后程序崩溃了。查看日志，抛出如下异常：

```
AndroidRuntime: FATAL EXCEPTION: Thread-176
    Process: com.example.joy.messagetest, PID: 11201
    android.view.ViewRootImpl$CalledFromWrongThreadException: Only the original thread that created a view hierarchy can touch its views.
        at android.view.ViewRootImpl.checkThread(ViewRootImpl.java:6357)
        at android.view.ViewRootImpl.requestLayout(ViewRootImpl.java:874)
```

这次终于看到了熟悉的错误日志，只有初始创建视图的线程才能触碰这些视图，也就是说只有主线程才能更新UI。通过下面一行

```
at android.view.ViewRootImpl.checkThread(ViewRootImpl.java:6357)
```

我们能发现点端倪：在 `framework/base/core/java/android/view/ViewRootImpl.java` 中有一个方法 `checkThread` ，源码如下：

```
void checkThread() {
    if (mThread != Thread.currentThread()) {
        throw new CalledFromWrongThreadException(
                "Only the original thread that created a view hierarchy can touch its views.");
    }
}
```

该异常就是在这里触发的。对于这个问题，如果你还想深入下去探究清楚，可以跟进去 RTFSC ! 这里推荐一篇文章，[Android中子线程真的不能更新UI吗？](https://www.cnblogs.com/xuyinhuan/p/5930287.html)

说了这么多，其实子线程是不能直接更新UI的。Android实现View更新有两组方法，分别是invalidate和postInvalidate。前者在UI线程中使用，后者在非UI线程即子线程中使用。换句话说，在子线程调用 invalidate 方法会导致线程不安全。熟悉View工作原理的人都知道，invalidate 方法会通知 view 立即重绘，刷新界面。作一个假设，现在我用 invalidate 在子线程中刷新界面，同时UI线程也在用 invalidate 刷新界面，这样会不会导致界面的刷新不能同步？这就是invalidate不能在子线程中使用的原因。

但是我们可以在子线程执行某段代码，需要更新UI的时候去通知主线程，让主线程来更新。如何做呢？常见的方法，除了前面提到的在UI线程创建Handler，在子线程发送消息到UI线程，通知UI线程更新UI，还有 handler.post(Runnable r)、 view.post(Runnable r)、activity.runOnUIThread(Runnable r)等方法。跟进去看源码，发现其实它们的实现原理都还是一样，最终都是通过Handler发送消息来实现的。下面分别用这几种方法实现一下在子线程更新UI。

修改后的布局文件代码如下：

```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:paddingBottom="@dimen/activity_vertical_margin"
    android:paddingLeft="@dimen/activity_horizontal_margin"
    android:paddingRight="@dimen/activity_horizontal_margin"
    android:paddingTop="@dimen/activity_vertical_margin"
    tools:context="com.example.joy.messagetest.MainActivity">

    <TextView
        android:id="@+id/tv_test"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_centerHorizontal="true"
        android:text="Hello World!" />

    <Button
        android:id="@+id/btn_test1"
        android:layout_below="@id/tv_test"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_centerHorizontal="true"
        android:text="子线程更新UI测试"/>

    <Button
        android:id="@+id/btn_test2"
        android:layout_below="@id/btn_test1"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_centerHorizontal="true"
        android:text="Handler发送消息"/>

    <Button
        android:id="@+id/btn_test3"
        android:layout_below="@id/btn_test2"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_centerHorizontal="true"
        android:text="Handler.Post"/>

    <Button
        android:id="@+id/btn_test4"
        android:layout_below="@id/btn_test3"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_centerHorizontal="true"
        android:text="View.Post"/>

    <Button
        android:id="@+id/btn_test5"
        android:layout_below="@id/btn_test4"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_centerHorizontal="true"
        android:text="Activity.RunOnUIThread"/>

</RelativeLayout>
```

java代码如下：

```
package com.example.joy.messagetest;

import android.os.AsyncTask;
import android.os.Handler;
import android.os.Looper;
import android.os.Message;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.view.View;
import android.widget.Button;
import android.widget.TextView;
import android.widget.Toast;

public class MainActivity extends AppCompatActivity implements View.OnClickListener {

    private TextView mTvTest;
    private Button mBtnTest1;
    private Button mBtnTest2;
    private Button mBtnTest3;
    private Button mBtnTest4;
    private Button mBtnTest5;


    private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            if (msg.what == 100) {
                mTvTest.setText("由Handler发送消息");
            }
        }
    };


    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        initView();

        new Thread(new Runnable() {
            @Override
            public void run() {
                mTvTest.setText("子线程可以更新UI");
            }
        }).start();
    }

    private void initView() {
        mTvTest = (TextView) findViewById(R.id.tv_test);
        mBtnTest1 = (Button) findViewById(R.id.btn_test1);
        mBtnTest2 = (Button) findViewById(R.id.btn_test2);
        mBtnTest3 = (Button) findViewById(R.id.btn_test3);
        mBtnTest4 = (Button) findViewById(R.id.btn_test4);
        mBtnTest5 = (Button) findViewById(R.id.btn_test5);
        mBtnTest1.setOnClickListener(this);
        mBtnTest2.setOnClickListener(this);
        mBtnTest2.setOnClickListener(this);
        mBtnTest3.setOnClickListener(this);
        mBtnTest4.setOnClickListener(this);
        mBtnTest5.setOnClickListener(this);
    }

    @Override
    public void onClick(View v) {
        switch (v.getId()) {
            case R.id.btn_test1:
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        mTvTest.setText("子线程真的可以更新UI吗？");
                    }
                }).start();
                break;
            case R.id.btn_test2:   //通过发送消息
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        mHandler.sendEmptyMessage(100);
                    }
                }).start();
                break;
            case R.id.btn_test3:  //通过Handler.post方法
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        mHandler.post(new Runnable() {
                            @Override
                            public void run() {
                                mTvTest.setText("handler.post");
                            }
                        });
                    }
                }).start();
                break;
            case R.id.btn_test4:  //通过 view.post方法
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        mTvTest.post(new Runnable() {
                            @Override
                            public void run() {
                                mTvTest.setText("view.post");
                            }
                        });
                    }
                }).start();
                break;
            case R.id.btn_test5:  //通过 activity 的 runOnUiThread方法
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        runOnUiThread(new Runnable() {
                            @Override
                            public void run() {
                                mTvTest.setText("runOnUIThread");
                            }
                        });
                    }
                }).start();
                break;
            default:
                break;
        }
    }
}
```

运行一下效果如下图：

![](/assets/images/技术/编程/Android/Android子线程更新UI的方法/pic3.gif)

以上就是消息机制最常见的应用场景——在子线程通知主线程更新UI的几种用法。