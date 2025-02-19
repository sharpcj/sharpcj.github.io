---
layout: post
title: "Android之AppWidget开发浅析"
date:  2017-02-04 18:58:12 +0800
categories: ["技术", "编程", "Android"]
tag: ["Android", "Widget", "小部件"]
---

## 什么是AppWidget
AppWidget 即桌面小部件，也叫桌面控件，就是能直接显示在Android系统桌面上的小程序，先看图：
![](/assets/images/技术/编程/Android/Android之AppWidget开发浅析/pic1.png)
![](/assets/images/技术/编程/Android/Android之AppWidget开发浅析/pic2.png)

图中我用黄色箭头指示的即为AppWidget，一些用户使用比较频繁的程序，可以做成AppWidget，这样能方便地使用。典型的程序有时钟、天气、音乐播放器等。AppWidget 是Android 系统应用开发层面的一部分，有着特殊用途，使用得当的化，的确会为app 增色不少，它的工作原理是把一个进程的控件嵌入到别外一个进程的窗口里的一种方法。长按桌面空白处，会出现一个 AppWidget 的文件夹，在里面找到相应的 AppWidget ，长按拖出，即可将 AppWidget 添加到桌面。

## 如何开发AppWidget
AppWidget 是通过 BroadCastReceiver 的形式进行控制的，开发 AppWidget 的主要类为 AppWidgetProvider, 该类继承自 BroadCastReceiver。为了实现桌面小部件，开发者只要开发一个继承自 AppWidgetProvider 的子类，并重写它的 onUpdate() 方法即可。重写该方法，一般来说可按如下几个步骤进行：
**1. 创建一个 RemoteViews 对象，这个对象加载时指定了桌面小部件的界面布局文件。**
**2. 设置 RemoteViews 创建时加载的布局文件中各个元素的属性。**
**3. 创建一个 ComponentName 对象**
**4. 调用 AppWidgetManager 更新桌面小部件。**

下面来看一个实际的例子，用 Android Studio 自动生成的例子来说。（注：我用的是最新版的 AS 2.2.3,下面简称 AS。）

新建了一个 HelloWorld 项目，然后新建一个 AppWidget ，命名为 MyAppWidgetProvider，按默认下一步，就完成了一个最简单的AppWidget的开发。运行程序之后，将小部件添加到桌面。操作步骤和默认效果如下：

![](/assets/images/技术/编程/Android/Android之AppWidget开发浅析/pic3.png)
![](/assets/images/技术/编程/Android/Android之AppWidget开发浅析/pic4.png)

我们看看 AS 为我们自动生成了哪些代码呢？对照着上面说的的步骤我们来看看。
首先，有一个 MyAppWidgetProvider 的类。

```
package com.example.joy.remoteviewstest;

import android.appwidget.AppWidgetManager;
import android.appwidget.AppWidgetProvider;
import android.content.Context;
import android.widget.RemoteViews;

/**
 * Implementation of App Widget functionality.
 */
public class MyAppWidgetProvider extends AppWidgetProvider {

    static void updateAppWidget(Context context, AppWidgetManager appWidgetManager,
                                int appWidgetId) {

        CharSequence widgetText = context.getString(R.string.appwidget_text);　　
        // Construct the RemoteViews object
        RemoteViews views = new RemoteViews(context.getPackageName(), R.layout.my_app_widget_provider);
        views.setTextViewText(R.id.appwidget_text, widgetText);

        // Instruct the widget manager to update the widget
        appWidgetManager.updateAppWidget(appWidgetId, views);
    }

    @Override
    public void onUpdate(Context context, AppWidgetManager appWidgetManager, int[] appWidgetIds) {
        // There may be multiple widgets active, so update all of them
        for (int appWidgetId : appWidgetIds) {
            updateAppWidget(context, appWidgetManager, appWidgetId);
        }
    }

    @Override
    public void onEnabled(Context context) {
        // Enter relevant functionality for when the first widget is created
    }

    @Override
    public void onDisabled(Context context) {
        // Enter relevant functionality for when the last widget is disabled
    }
}
```

该类继承自 AppWidgetProvider ，AS默认帮我们重写 onUpdate() 方法，遍历 appWidgetIds, 调用了 updateAppWidget() 方法。再看 updateAppWidget() 方法，很简单，只有四行：

第一行，`CharSequence widgetText = context.getString(R.string.appwidget_text);`声明了一个字符串；
第二行，`RemoteViews views = new RemoteViews(context.getPackageName(), R.layout.my_app_widget_provider);`创建了一个 RemoteViews 对象,第一个参数传应用程序包名，第二个参数指定了，RemoteViews 加载的布局文件。这一行对应上面步骤中说的第一点。可以看到在 res/layout/ 目录下面 AS 自动生成了一个 my_app_widget_provider.xml 文件，内容如下：

```
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="#09C"
    android:padding="@dimen/widget_margin">

    <TextView
        android:id="@+id/appwidget_text"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_centerHorizontal="true"
        android:layout_centerVertical="true"
        android:layout_margin="8dp"
        android:background="#09C"
        android:contentDescription="@string/appwidget_text"
        android:text="@string/appwidget_text"
        android:textColor="#ffffff"
        android:textStyle="bold|italic" />

</RelativeLayout>
```

这个文件就是我们最后看到的桌面小部件的样子，布局文件中只有一个TextView。这是你可能会问，想要加图片可以吗？可以，就像正常的Activity布局一样添加 ImageView 就行了，聪明的你可能开始想自定义小部件的样式了，添加功能强大外观漂亮逼格高的自定义控件了，很遗憾，不可以。小部件布局文件可以添加的组件是有限制的，详细内容在下文介绍RemoteViews 时再说。

第三行，`views.setTextViewText(R.id.appwidget_text, widgetText);`将第一行声明的字符串赋值给上面布局文件中的 TextView,注意这里赋值时，指定TextView的 id，要对应起来。这一行对于了上面步骤中的第二点。
第四行, `appWidgetManager.updateAppWidget(appWidgetId, views);`这里调用了 `appWidgetManager.updateAppWidget()` 方法，更新小部件。这一行对应了上面步骤中的第四点。

这时，你可能有疑问了，上面明明说了四个步骤，其中第三步，创建一个 ComponentName 对象，明明就不需要。的确，这个例子中也没有用到。如果我们手敲第四步代码，AS的智能提示会告诉你，`appWidgetManager.updateAppWidget()` 有三个重载的方法。源码中三个方法没有写在一起，为了方便，这里我复制贴出官方 API 中的介绍:

| 返回值 | 方法签名及说明 |
| ----------- | ----------- |
| void | `updateAppWidget(ComponentName provider, RemoteViews views)` Set the RemoteViews to use for all AppWidget instances for the supplied AppWidget provider. |
| void | `updateAppWidget(int[] appWidgetIds, RemoteViews views)` set the RemoteViews to use for the specified appWidgetIds. |
| void | `updateAppWidget(int appWidgetId, RemoteViews views)` Set the RemoteViews to use for the specified appWidgetId. |

这个三个方法都接收两个参数，第二个参数都是 RemoteViews 对象。其中第一个方法的第一个参数就是 ComponentName 对象，更新所有的 AppWidgetProvider 提供的所有的 AppWidget 实例，第二个方法时更新明确指定 Id 的 AppWidget 的对象集，第三个方法，更新明确指定 Id 的某个 AppWidget 对象。所以一般我们使用第一个方法，针对所有的 AppWidget 对象，我们也可以根据需要选择性地去更新。

到这里，所有步骤都结束了，就完了？还没。前面说了，自定义的 MyAppWidgetProvider 继承自 AppWidgetProvider，而 AppWidgetProvider 又是继承自 BroadCastReceiver，

所以说 MyAppWidgetProvider 本质上是一个广播接受者，属于四大组件之一，需要我们的清单文件中注册。打开AndroidManifest.xml文件可以看到，的确是注册了小部件的，内容如下：

```
<receiver android:name=".MyAppWidgetProvider">
  　　 <intent-filter>
     　　　　<action android:name="android.appwidget.action.APPWIDGET_UPDATE" />
      </intent-filter>

      <meta-data
            android:name="android.appwidget.provider"
            android:resource="@xml/my_app_widget_provider_info" />
  </receiver>
```

上面代码中有一个 Action，这个 Action 必须要加，且不能更改，属于系统规范，是作为小部件的标识而存在的。如果不加，这个 Receiver 就不会出现在小部件列表里面。然后看到小部件指定了 @xml/my_app_widget_provider_info 作为meta-data，细心的你发现了，在 res/ 目录下面建立了一个 xml 文件夹，下面新建了一个 my_app_widget_provider_info.xml 文件，内容如下：

```
<?xml version="1.0" encoding="utf-8"?>
<appwidget-provider xmlns:android="http://schemas.android.com/apk/res/android"
    android:initialKeyguardLayout="@layout/my_app_widget_provider"
    android:initialLayout="@layout/my_app_widget_provider"
    android:minHeight="40dp"
    android:minWidth="40dp"
    android:previewImage="@drawable/example_appwidget_preview"
    android:resizeMode="horizontal|vertical"
    android:updatePeriodMillis="86400000"
    android:widgetCategory="home_screen">
</appwidget-provider>
```

这里配置了一些小部件的基本信息，常用的属性有 initialLayout 就是小部件的初始化布局， minHeight 定义了小部件的最小高度，previewImage 指定了小部件在小部件列表里的预览图，updatePeriodMillis 指定了小部件更新周期，单位为毫秒。更多属性，可以查看API文档。

到这里，上面这个极简单的小部件开发过程就真的结束了。为了开发出更强大一点小部件，我们还需要进一步了解 RemoteViews 和 AppWidgetProvider。

## AppWidget的妆容——RemoteViews
下面简单说说 RemoteViews 相关的几个类。

### RemoteViews
RemoteViews，从字面意思理解为它是一个远程视图。是一种远程的 View，它在其它进程中显示，却可以在另一个进程中更新。RemoteViews 在Android中的使用场景主要有：自定义通知栏和桌面小部件。

在RemoteViews 的构造函数中，第二个参数接收一个 layout 文件来确定 RemoteViews 的视图；然后，我们调用RemoteViews 中的 set 方法对 layout 中的各个组件进行设置，例如，可以调用 setTextViewText() 来设置 TextView 组件的文本。

前面提到，小部件布局文件可以添加的组件是有限制的，它可以支持的 View 类型包括四种布局：FrameLayout、LinearLayout、RelativeLayout、GridLayout 和 13 种View: AnalogClock、Button、Chronometer、ImageButton、ImageView、ProgressBar、TextView、ViewFlipper、ListView、GridView、StackView、AdapterViewFlipper、ViewSub。注意：RemoteViews 也并不支持上述 View 的子类。

RemoteViews 提供了一系列 setXXX() 方法来为小部件的子视图设置属性。具体可以参考 API 文档。

### RemoteViewsService
RemoteViewsService，是管理RemoteViews的服务。一般，当AppWidget 中包含 GridView、ListView、StackView 等集合视图时，才需要使用RemoteViewsService来进行更新、管理。RemoteViewsService 更新集合视图的一般步骤是：

1. 通过 setRemoteAdapter() 方法来设置 RemoteViews 对应 RemoteViewsService 。

2. 之后在 RemoteViewsService 中，实现 RemoteViewsFactory 接口。然后，在 RemoteViewsFactory 接口中对集合视图的各个子项进行设置，例如 ListView 中的每一Item。

### RemoteViewsFactory
通过RemoteViewsService中的介绍，我们知道RemoteViewsService是通过 RemoteViewsFactory来具体管理layout中集合视图的，RemoteViewsFactory是RemoteViewsService中的一个内部接口。RemoteViewsFactory提供了一系列的方法管理集合视图中的每一项。例如：

```
RemoteViews getViewAt(int position)
```

通过getViewAt()来获取“集合视图”中的第position项的视图，视图是以RemoteViews的对象返回的。

```
int getCount()
```

通过getCount()来获取“集合视图”中所有子项的总数。

## AppWidget的美貌——AppWidgetProvider
我们说一位女同事漂亮，除了因为她穿的衣服、化的妆漂亮以外，我想最主要的原因还是她本人长的漂亮吧。同样，小部件之所以有附着在桌面，跨进程更新 View 的能力，主要是因为 `AppWidgetProvider` 是一个广播接收者。我们发现，上面的例子中，AS 帮我们自动生成的代码中，除了 onUpdate() 方法被我们重写了，还有重写 onEnable() 和 onDisable() 两个方法，但都是空实现，这两个方法什么时候会被调用？还有，我们说自定义的 MyAppWidgetProvider，继承自 AppWidgetProvider，而 MyAppWidgetProvider 又是BroadCastReceiver 的子类，而我们却没有向写常规广播接收者一样重写 onReceiver() 方法？下面跟进去 AppWidgetProvider 源码，一探究竟。

这个类代码并不多，其实，AppWidgetProvider 出去构造方法外，总共只有下面这些方法：

- **onEnable():** 当小部件第一次被添加到桌面时回调该方法，可添加多次，但只在第一次调用。对用广播的 Action 为 ACTION_APPWIDGET_ENABLE。

- **onUpdate():**  当小部件被添加时或者每次小部件更新时都会调用一次该方法，配置文件中配置小部件的更新周期 updatePeriodMillis，每次更新都会调用。对应广播 Action 为：ACTION_APPWIDGET_UPDATE 和 ACTION_APPWIDGET_RESTORED 。

- **onDisabled():** 当最后一个该类型的小部件从桌面移除时调用，对应的广播的 Action 为 ACTION_APPWIDGET_DISABLED。

- **onDeleted():** 每删除一个小部件就调用一次。对应的广播的 Action 为： ACTION_APPWIDGET_DELETED 。

- **onRestored():** 当小部件从备份中还原，或者恢复设置的时候，会调用，实际用的比较少。对应广播的 Action 为 ACTION_APPWIDGET_RESTORED。

- **onAppWidgetOptionsChanged():** 当小部件布局发生更改的时候调用。对应广播的 Action 为 ACTION_APPWIDGET_OPTIONS_CHANGED。

- **onReceive():** AppWidgetProvider 重写了该方法，用于分发具体的时间给上述的方法。看看源码：

```
public void onReceive(Context context, Intent intent) {
        // Protect against rogue update broadcasts (not really a security issue,
        // just filter bad broacasts out so subclasses are less likely to crash).
        String action = intent.getAction();
        if (AppWidgetManager.ACTION_APPWIDGET_UPDATE.equals(action)) {
            Bundle extras = intent.getExtras();
            if (extras != null) {
                int[] appWidgetIds = extras.getIntArray(AppWidgetManager.EXTRA_APPWIDGET_IDS);
                if (appWidgetIds != null && appWidgetIds.length > 0) {
                    this.onUpdate(context, AppWidgetManager.getInstance(context), appWidgetIds);
                }
            }
        } else if (AppWidgetManager.ACTION_APPWIDGET_DELETED.equals(action)) {
            Bundle extras = intent.getExtras();
            if (extras != null && extras.containsKey(AppWidgetManager.EXTRA_APPWIDGET_ID)) {
                final int appWidgetId = extras.getInt(AppWidgetManager.EXTRA_APPWIDGET_ID);
                this.onDeleted(context, new int[] { appWidgetId });
            }
        } else if (AppWidgetManager.ACTION_APPWIDGET_OPTIONS_CHANGED.equals(action)) {
            Bundle extras = intent.getExtras();
            if (extras != null && extras.containsKey(AppWidgetManager.EXTRA_APPWIDGET_ID)
                    && extras.containsKey(AppWidgetManager.EXTRA_APPWIDGET_OPTIONS)) {
                int appWidgetId = extras.getInt(AppWidgetManager.EXTRA_APPWIDGET_ID);
                Bundle widgetExtras = extras.getBundle(AppWidgetManager.EXTRA_APPWIDGET_OPTIONS);
                this.onAppWidgetOptionsChanged(context, AppWidgetManager.getInstance(context),
                        appWidgetId, widgetExtras);
            }
        } else if (AppWidgetManager.ACTION_APPWIDGET_ENABLED.equals(action)) {
            this.onEnabled(context);
        } else if (AppWidgetManager.ACTION_APPWIDGET_DISABLED.equals(action)) {
            this.onDisabled(context);
        } else if (AppWidgetManager.ACTION_APPWIDGET_RESTORED.equals(action)) {
            Bundle extras = intent.getExtras();
            if (extras != null) {
                int[] oldIds = extras.getIntArray(AppWidgetManager.EXTRA_APPWIDGET_OLD_IDS);
                int[] newIds = extras.getIntArray(AppWidgetManager.EXTRA_APPWIDGET_IDS);
                if (oldIds != null && oldIds.length > 0) {
                    this.onRestored(context, oldIds, newIds);
                    this.onUpdate(context, AppWidgetManager.getInstance(context), newIds);
                }
            }
        }
    }
```

## AppWidget 练习
下面再自己写个例子，学习 RemoteViews 中的其它知识点，这个例子中小部件布局中用到 button 和 listview。上代码：

小部件的布局文件 `mul_app_widget_provider.xml` 如下：

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="horizontal"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <LinearLayout
        android:layout_width="100dp"
        android:layout_height="200dp"
        android:orientation="vertical">
        <ImageView
            android:id="@+id/iv_test"
            android:layout_width="match_parent"
            android:layout_height="100dp"
            android:src="@mipmap/ic_launcher"/>
        <Button
            android:id="@+id/btn_test"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="点击跳转"/>
    </LinearLayout>

    <TextView
        android:layout_width="1dp"
        android:layout_height="200dp"
        android:layout_marginLeft="5dp"
        android:layout_marginRight="5dp"
        android:background="#f00"/>

    <ListView
        android:id="@+id/lv_test"
        android:layout_width="100dp"
        android:layout_height="200dp">
    </ListView>

</LinearLayout>
```

小部件的配置信息 `mul_app_widget_provider_info.xml` 如下：

```
<?xml version="1.0" encoding="utf-8"?>
<appwidget-provider xmlns:android="http://schemas.android.com/apk/res/android"
    android:initialLayout="@layout/mul_app_widget_provider"
    android:minHeight="200dp"
    android:minWidth="200dp"
    android:previewImage="@mipmap/a1"
    android:updatePeriodMillis="86400000">
</appwidget-provider>
```

MulAppWidgetProvider.java：

```
package com.example.joy.remoteviewstest;

import android.app.PendingIntent;
import android.appwidget.AppWidgetManager;
import android.appwidget.AppWidgetProvider;
import android.content.ComponentName;
import android.content.Context;
import android.content.Intent;
import android.os.Bundle;
import android.text.TextUtils;
import android.widget.RemoteViews;
import android.widget.Toast;

public class MulAppWidgetProvider extends AppWidgetProvider {

    public static final String CHANGE_IMAGE = "com.example.joy.action.CHANGE_IMAGE";

    private RemoteViews mRemoteViews;
    private ComponentName mComponentName;

    private int[] imgs = new int[]{
            R.mipmap.a1,
            R.mipmap.b2,
            R.mipmap.c3,
            R.mipmap.d4,
            R.mipmap.e5,
            R.mipmap.f6
    };


    @Override
    public void onUpdate(Context context, AppWidgetManager appWidgetManager, int[] appWidgetIds) {
        mRemoteViews = new RemoteViews(context.getPackageName(), R.layout.mul_app_widget_provider);
        mRemoteViews.setImageViewResource(R.id.iv_test, R.mipmap.ic_launcher);
        mRemoteViews.setTextViewText(R.id.btn_test, "点击跳转到Activity");
        Intent skipIntent = new Intent(context, MainActivity.class);
        PendingIntent pi = PendingIntent.getActivity(context, 200, skipIntent, PendingIntent.FLAG_CANCEL_CURRENT);
        mRemoteViews.setOnClickPendingIntent(R.id.btn_test, pi);

        // 设置 ListView 的adapter。
        // (01) intent: 对应启动 ListViewService(RemoteViewsService) 的intent
        // (02) setRemoteAdapter: 设置 ListView 的适配器
        // 通过setRemoteAdapter将 ListView 和ListViewService关联起来，
        // 以达到通过 GridWidgetService 更新 gridview 的目的
        Intent lvIntent = new Intent(context, ListViewService.class);
        mRemoteViews.setRemoteAdapter(R.id.lv_test, lvIntent);
        mRemoteViews.setEmptyView(R.id.lv_test,android.R.id.empty);

        // 设置响应 ListView 的intent模板
        // 说明：“集合控件(如GridView、ListView、StackView等)”中包含很多子元素，如GridView包含很多格子。
        // 它们不能像普通的按钮一样通过 setOnClickPendingIntent 设置点击事件，必须先通过两步。
        // (01) 通过 setPendingIntentTemplate 设置 “intent模板”，这是比不可少的！
        // (02) 然后在处理该“集合控件”的RemoteViewsFactory类的getViewAt()接口中 通过 setOnClickFillInIntent 设置“集合控件的某一项的数据”
        
        /*
         * setPendingIntentTemplate 设置pendingIntent 模板
         * setOnClickFillInIntent   可以将fillInIntent 添加到pendingIntent中
         */
        Intent toIntent = new Intent(CHANGE_IMAGE);
        PendingIntent pendingIntent = PendingIntent.getBroadcast(context, 200, toIntent, PendingIntent.FLAG_UPDATE_CURRENT);
        mRemoteViews.setPendingIntentTemplate(R.id.lv_test, pendingIntent);


        mComponentName = new ComponentName(context, MulAppWidgetProvider.class);
        appWidgetManager.updateAppWidget(mComponentName, mRemoteViews);
    }

    @Override
    public void onReceive(Context context, Intent intent) {
        super.onReceive(context, intent);
        if(TextUtils.equals(CHANGE_IMAGE,intent.getAction())){
            Bundle extras = intent.getExtras();
            int position = extras.getInt(ListViewService.INITENT_DATA);
            mRemoteViews = new RemoteViews(context.getPackageName(), R.layout.mul_app_widget_provider);
            mRemoteViews.setImageViewResource(R.id.iv_test, imgs[position]);
            mComponentName = new ComponentName(context, MulAppWidgetProvider.class);
            AppWidgetManager.getInstance(context).updateAppWidget(mComponentName, mRemoteViews);
        }
    }
}
```

MainActivity.java：

```
package com.example.joy.remoteviewstest;

import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;

public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }
}
```

下面重点是 ListView 在小部件中的用法：

```
package com.example.joy.remoteviewstest;

import android.content.Context;
import android.content.Intent;
import android.os.Bundle;
import android.widget.RemoteViews;
import android.widget.RemoteViewsService;

import java.util.ArrayList;
import java.util.List;

public class ListViewService extends RemoteViewsService {
    public static final String INITENT_DATA = "extra_data";

    @Override
    public RemoteViewsFactory onGetViewFactory(Intent intent) {
        return new ListRemoteViewsFactory(this.getApplicationContext(), intent);
    }

    private class ListRemoteViewsFactory implements RemoteViewsService.RemoteViewsFactory {

        private Context mContext;

        private List<String> mList = new ArrayList<>();

        public ListRemoteViewsFactory(Context context, Intent intent) {
            mContext = context;
        }

        @Override
        public void onCreate() {
            mList.add("一");
            mList.add("二");
            mList.add("三");
            mList.add("四");
            mList.add("五");
            mList.add("六");
        }

        @Override
        public void onDataSetChanged() {

        }

        @Override
        public void onDestroy() {
            mList.clear();
        }

        @Override
        public int getCount() {
            return mList.size();
        }

        @Override
        public RemoteViews getViewAt(int position) {
            RemoteViews views = new RemoteViews(mContext.getPackageName(), android.R.layout.simple_list_item_1);
            views.setTextViewText(android.R.id.text1, "item:" + mList.get(position));

            Bundle extras = new Bundle();
            extras.putInt(ListViewService.INITENT_DATA, position);
            Intent changeIntent = new Intent();
            changeIntent.setAction(MulAppWidgetProvider.CHANGE_IMAGE);
            changeIntent.putExtras(extras);

            /* android.R.layout.simple_list_item_1 --- id --- text1
             * listview的item click：将 changeIntent 发送，
             * changeIntent 它默认的就有action 是provider中使用 setPendingIntentTemplate 设置的action*/
            views.setOnClickFillInIntent(android.R.id.text1, changeIntent);
            return views;
        }

        /* 在更新界面的时候如果耗时就会显示 正在加载... 的默认字样，但是你可以更改这个界面
         * 如果返回null 显示默认界面
         * 否则 加载自定义的，返回RemoteViews
         */
        @Override
        public RemoteViews getLoadingView() {
            return null;
        }

        @Override
        public int getViewTypeCount() {
            return 1;
        }

        @Override
        public long getItemId(int position) {
            return position;
        }

        @Override
        public boolean hasStableIds() {
            return false;
        }
    }
}
```

最后看看清单文件：

```
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.joy.remoteviewstest">

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <receiver android:name=".MulAppWidgetProvider"
            android:label="@string/app_name">
            <intent-filter>
                <action android:name="com.example.joy.action.CHANGE_IMAGE"/>
                <action android:name="android.appwidget.action.APPWIDGET_UPDATE"/>
            </intent-filter>
            <meta-data
                android:name="android.appwidget.provider"
                android:resource="@xml/mul_app_widget_provider_info">
            </meta-data>
        </receiver>

        <service android:name=".ListViewService"
            android:permission="android.permission.BIND_REMOTEVIEWS"
            android:exported="false"
            android:enabled="true"/>

    </application>

</manifest>
```

这个小部件添加到桌面后有一个 ImageView 显示小机器人，下面有一个 Button ,右边有一个ListView。
这里主要看看，Button 和 ListView 在 RemoteViews中如何使用。
Button 设置 Text 和 TextView 一样，因为 Button 本身继承自 TextView，Button 设置点击事件如下：

```
Intent skipIntent = new Intent(context, MainActivity.class);
PendingIntent pi = PendingIntent.getActivity(context, 200, skipIntent, PendingIntent.FLAG_CANCEL_CURRENT);
mRemoteViews.setOnClickPendingIntent(R.id.btn_test, pi);
```

用到方法 `setOnClickPendingIntent`，PendingIntent 表示延迟的 Intent ， 与通知中的用法一样。这里点击之后跳转到了 MainActivity。
关于 ListView 的用法就复杂一些了。首先需要自定义一个类继承自 RemoteViewsServices ，并重写 onGetViewFactory 方法，返回 `RemoteViewsService.RemoteViewsFactory` 接口的对象。这里定义了一个内部类实现该接口，需要重写多个方法，与 ListView 的多布局适配很类似。重点方法是 `public RemoteViews getViewAt(int position){}` 这个方法中指定了 ListView 的每一个 item 的布局以及内容，同时通过  setOnClickFillInIntent() 或者 setOnClickPendingIntent() 给 item 设置点击事件。这里我实现的点击 item,替换左边的 ImageView 的图片。重写了 MulAppWidgetProvider 类的 onReceiver 方法，处理替换图片的逻辑。

程序运行效果如下图：

![](/assets/images/技术/编程/Android/Android之AppWidget开发浅析/pic5.gif)



