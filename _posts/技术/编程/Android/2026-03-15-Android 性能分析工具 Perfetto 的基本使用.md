---
layout: post
title: "Android 性能分析工具 Perfetto 的基本使用"
date:  2026-03-15 13:58:12 +0800
categories: ["技术", "编程", "Android"]
tag: ["Android", "性能分析", "perfetto"]
image:
  path: /assets/images/技术/编程/Android/Android 性能分析工具 perfetto 的基本使用/title.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: perfetto
---

## 一、Android 性能分析工具
Android 性能分析工具有很多，针对不同的问题场景，选择不同的工具。针对卡顿相关问题的分析，常用的有 TraceView、SysTrace、Perfetto 以及 Android Studio 中集成的 Android Profiler 。

TraceView：一个函数级的性能分析器。它通过“插桩”（Instrumentation）技术，在每个方法的入口和出口插入计时代码，从而精确记录每个函数的调用耗时和调用次数。它能深入到代码的具体方法，它本身对应用性能影响巨大，现在基本已经被弃用。

SysTrace：一个系统级的性能分析工具。它主要依赖 Linux 内核的 ftrace 机制，从系统层面收集 Android OS 和应用程序的运行时信息，如 CPU 调度、Binder 通信、图形渲染等。它关注的是系统各组件间的协作和整体性能表现。

Perfetto：Android 官方的最新的性能分析平台，旨在完全取代 Systrace。它不仅兼容 Systrace 的所有功能，还提供了更强大的数据收集能力（覆盖 Android, Linux, Chrome 等）、更灵活的配置、更低的开销以及更先进的分析界面（支持 SQL 查询）。

Android Profiler：集成在 Android Studio 中的一体化性能分析工具。它并非一个独立的底层技术，而是一个图形化界面，整合了多种分析能力。在新版 Android Studio 中，其底层主要依赖 Perfetto 来提供 CPU、内存、网络和能耗的分析数据。

综上，当前 Android 开发中，日常开发中，对于简单的性能问题，可以借助 Android Profiler 进行分析，方便便捷。对于复杂的性能问题，尤其是涉及到系统层的问题，则 Perfetto 是更合适的工具。本文主要介绍 Perfetto 的基本使用。

## 二、Perfetto 如何抓取 Trace
### 2.1 使用命令行抓取

**基础命令行格式**

```
adb shell perfetto -o /data/misc/perfetto-traces/trace_file.perfetto-trace -t <时长> <数据源列表>
```

-o：指定 trace 文件在手机上的输出路径。
-t：指定抓取的时长，例如 10s 代表 10 秒。

示例：以下命令会抓取 20 秒的 trace，包含 CPU 调度、频率、应用生命周期、窗口管理、图形、View 绘制、Binder 驱动等关键数据，并保存在手机上：

```
// 1. 首先执行命令
adb shell perfetto -o /data/misc/perfetto-traces/trace_file.perfetto-trace -t 20s \
sched freq idle am wm gfx view binder_driver hal dalvik camera input res memory

// 2. 操作手机，复现场景，比如滑动或者启动等

// 3. 将 trace 文件 pull 到本地
adb pull /data/misc/perfetto-traces/trace_file.perfetto-trace
```

**极简全量抓取**
如果想抓取所有可用数据源，可以直接使用 -a 参数，无需手动列出模块：

```
// 抓取 10 秒所有数据
adb shell perfetto -o /data/misc/perfetto-traces/trace.perfetto-trace -t 10s -a "com.xxx.xxx"
```

**带配置文件抓取**
当需要更精细地控制抓取行为，例如指定缓冲区大小、抓取特定应用、自定义 ftrace 事件或进行长时间记录时，推荐使用配置文件。Perfetto 使用文本格式的配置文件（.pbtx）。
1. 编写一个配置文件：
创建一个名为 trace_config.pbtx 的文本文件，内容如下示例：

```
# 抓取时长，单位毫秒
duration_ms: 30000

# 定义缓冲区大小和策略
buffers: {
  size_kb: 65536  # 缓冲区大小
  fill_policy: DISCARD  # 满了则丢弃新数据
}

# 定义数据源
data_sources: {
  config {
    name: "linux.ftrace"
    ftrace_config {
      # ftrace 事件
      ftrace_events: "sched/sched_switch"
      ftrace_events: "power/suspend_resume"
      # atrace 类别
      atrace_categories: "gfx"
      atrace_categories: "view"
      # 指定要跟踪的应用包名
      atrace_apps: "com.example.app"
    }
  }
}

# 收集进程统计信息
data_sources: {
  config {
    name: "linux.process_stats"
    process_stats_config {
      scan_all_processes_on_start: true
    }
  }
}
```

2. 执行抓取命令
将配置文件推送到手机并执行。根据 Android 版本的不同，命令略有差异。
- Android 12 及以上版本：支持直接通过 -c 参数指定配置文件路径。通常使用 `/data/misc/perfetto-configs` 目录来存储配置文件

```
# 推送配置文件
adb push trace_config.pbtx /data/misc/perfetto-configs/trace_config.pbtx

# 执行抓取
adb shell perfetto -c /data/misc/perfetto-configs/trace_config.pbtx -o /data/misc/perfetto-traces/trace.pftrace
```

- Android 12 以下版本：需要使用 cat 命令将配置文件内容通过管道传递给 perfetto。

```
# 推送配置文件
adb push trace_config.pbtx /data/local/tmp/

# 执行抓取 (使用 -c - 从标准输入读取配置)
adb shell 'cat /data/local/tmp/trace_config.pbtx | perfetto --txt -c - -o /data/misc/perfetto-traces/trace.pftrace'
```

**使用 Perfetto 官方提供的脚本抓取**
Perfetto 团队还提供了一个便捷的脚本tools/record_android_trace，它简化了从命令行记录跟踪的流程。这个脚本会自动处理路径问题，完成跟踪后自动拉取跟踪文件，并在浏览器中打开它。本质上这个脚本还是使用的 adb shell perfetto 命令，不过官方帮你封装好了,使用示例：
在Linux和Mac上：

```
curl -O https://raw.githubusercontent.com/google/perfetto/master/tools/record_android_trace
chmod u+x record_android_trace

./record_android_trace -o trace_file.perfetto-trace -t 10s -b 32mb sched gfx wm
```

在Windows上：

```
curl -O https://raw.githubusercontent.com/google/perfetto/master/tools/record_android_trace
python3 record_android_trace -o trace_file.perfetto-trace -t 10s -b 32mb sched gfx wm
```

用命令行抓取的之后，生成的 trace 文件，在 [ui.perfetto.dev](https://ui.perfetto.dev) 上打开进行查看分析。使用官方脚本抓取之后，会直接帮你直接打开。强烈推荐使用该方法，一步到位。

**其它抓取方法**
另外还有一些其它抓取 trace 的方法，使用手机上的开发者工具抓取，在开发者模式中启动系统跟踪应用，或者在 Perfetto 网页上连接设备进行抓取。这些方法实际操作起来都不是很方便，不是很推荐，感兴趣的可以研究一下。

## 三、如何利用 Perfetto 分析 Trace 文件
Android 整个渲染流程很复杂，涉及到知识点很多，这里只是介绍如何查看 Trace 文件。

### 3.1 Frame Timeline
Perfetto 相比 Systrace 的一个重要优势是提供了 FrameTimeline 功能，可以一眼就可以看到卡顿的地方。需要 Android 12 或者更高版本支持。
当帧在屏幕上的实际呈现时间与调度器预期的呈现时间不匹配时，就会产生卡顿。FrameTimeline 为每个有帧在屏幕上显示的应用添加了两个新的轨道：

![](/assets/images/技术/编程/Android/Android%20性能分析工具%20perfetto%20的基本使用/time.png)

FrameTimeline 使用直观的颜色来标识不同的帧状态:

| 颜色   | 含义           | 说明                                                                 |
|--------|----------------|----------------------------------------------------------------------|
| 绿色   | 正常帧         | 没有观察到卡顿，理想状态                                             |
| 浅绿色 | 高延迟状态     | 帧率稳定但帧呈现延迟，导致输入延迟增加                               |
| 红色   | 卡顿帧         | 当前进程导致的卡顿                                                   |
| 黄色   | 应用无责任卡顿 | 帧出现卡顿但应用不是原因，SurfaceFlinger 导致的卡顿                  |
| 蓝色   | 丢帧           | SurfaceFlinger 丢弃了该帧，选择了更新的帧                            |

### 3.2 主线程和渲染线程的工作流程
一张图道明：

![](/assets/images/技术/编程/Android/Android%20性能分析工具%20perfetto%20的基本使用/thread.png)

